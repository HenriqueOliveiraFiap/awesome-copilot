---
title: 'ADR-0001: M2M - Autenticação e Autorização entre Serviços Internos com AWS Cognito - V3'
status: 'Em revisão'
date: '2026-04-08'
authors:
  - 'Henrique Oliveira'
revisores:
  - 'Tecnologia'
  - 'Plataforma'
  - 'DevOps'
  - 'Segurança'
tags:
  - 'autenticação'
  - 'autorização'
  - 'm2m'
  - 'aws-cognito'
  - 'oauth2'
  - 'microsserviços'
  - 'segurança'
supersedes: 'ADR-v2: M2M - Autenticação e Autorização entre Serviços Internos'
superseded_by: ''
---

# [ADR] M2M - Autenticação e autorização entre serviços internos - V3

| **Campo** | **Valor** |
| --- | --- |
| **Status** | Em revisão |
| **Data** | 2026-04-08 |
| **Autores** | Henrique Oliveira |
| **Revisores** | Tecnologia, Plataforma, DevOps e Segurança |

---

## **Contexto e problema**

Atualmente, a comunicação entre os microsserviços da Aarin utiliza um token que pode ser compartilhado entre serviços, onde um único token tem capacidade para acessar diversas APIs.

O fluxo de criação de usuários no Gatekeeper/Keycloak funciona por exclusão de permissões:

- Um novo serviço criado recebe automaticamente todos os escopos disponíveis (todas as roles)
- Para restringir acesso, seria necessário **remover manualmente** os escopos desnecessários

Na prática operacional, múltiplos serviços utilizam **as mesmas credenciais** para autenticação:

- pix-api, kyc-api, fee-api podem estar usando o mesmo usuário/token
- Logs não permitem rastreabilidade: Impossível identificar qual serviço específico executou uma ação

Este modelo apresenta os seguintes problemas:

- **Violação do princípio do menor privilégio**: Um único token concede acesso irrestrito a todos os serviços da malha
- **Ponto único de falha de segurança**: O vazamento ou comprometimento do token expõe toda a infraestrutura
- **Ausência de rastreabilidade**: Impossibilidade de identificar qual serviço está acessando qual recurso
- **Falta de granularidade**: Não é possível restringir acesso a rotas ou operações específicas
- **Dificuldade de revogação**: Revogar acesso de um serviço específico afeta todos os demais

É necessário implementar uma solução de autenticação e autorização M2M que garanta isolamento de segurança, rastreabilidade, controle de acesso granular e que seja alinhada com padrões de mercado (OAuth 2.0). Lembrando que é necessário garantir que tudo isso seja implementado com baixa complexidade para que os serviços consumidores e consumidos tenham o mínimo de esforço para utilizar essa nova arquitetura, garantindo que seja fácil integrar e gerenciar os acessos entre serviços, evitando assim que se torne um gargalo para a escalabilidade.

---

## Solução

Adotar **AWS Cognito** como IdP (Identity Provider) dedicado para comunicação M2M, implementando o fluxo **OAuth 2.0 Client Credentials Grant** para autenticação entre serviços internos.

A solução utiliza controle de acesso baseado em **escopos**, onde cada endpoint é protegido por um ou mais escopos, e cada serviço consumidor possui apenas os escopos estritamente necessários para suas operações.

**A validação de tokens JWT será feita via código** em cada serviço (.NET, Node.js) utilizando middleware padrão, com suporte de **bibliotecas internas** que abstraem a complexidade de obtenção, cache e renovação de tokens.

### Decisão: Por que AWS Cognito?

Durante a análise de alternativas, foram considerados outros provedores de identidade (Auth0, Okta, manutenção do Keycloak). A escolha pelo AWS Cognito se deu pelos seguintes motivos:

- **Já utilizado na Aarin**: o Cognito é o IdP atual para usuários finais (BaaS, BaaS Admin), o que significa familiaridade da equipe, infraestrutura já provisionada e curva de aprendizado reduzida
- **Ecossistema AWS nativo**: integração direta com Secrets Manager, API Gateway Authorizer, VPC Endpoints, CloudWatch e IAM — sem dependência de SaaS externo de terceiros
- **Precificação M2M atualizada**: a AWS removeu o custo fixo de $6 por App Client; o custo ocorre apenas na geração de tokens, o que favorece o modelo de 1 App Client por serviço
- **VPC Endpoint Privado**: permite emissão de tokens sem tráfego de internet, reduzindo latência e superfície de ataque
- **Suporte a Client Credentials Grant**: fluxo OAuth 2.0 nativo para M2M, sem necessidade de adaptações

### Decisão: Por que validação no código e não Istio?

**Istio Service Mesh** foi cogitado no passado para implementação de controle de acesso direto na camada de rede, no entanto, foi **desconsiderado pela complexidade de implementação** e algo que não precisamos para este momento inicial.

**Para a Fase 1**, seguiremos com **validação JWT via código** (middleware padrão), que resolve 80% dos problemas com 20% da complexidade. Isto também facilita uma eventual migração futura para Istio, pois já teremos:

- Cognito configurado
- Bibliotecas internas prontas
- Modelo de scopes validado
- Times treinados em OAuth 2.0 e novas bibliotecas

A solução é composta por **3 pilares principais**:

---

## **1. AWS Cognito (Provedor de Identidade)**

O Cognito atua como autoridade central de identidade para todos os serviços internos:

### **User Pool M2M dedicado**

- Exclusivo para comunicação entre serviços (separado de usuários finais como BaaS ou BaaS Admin)
- Domínio OAuth habilitado para emissão de tokens via Client Credentials Grant
- **Isolamento total**: nenhum usuário humano tem acesso a este pool

### **VPC Endpoint Privado**

- Permite gerar tokens **sem precisar ir para a internet**
- Comunicação interna via VPC reduz latência e aumenta segurança
- ⚠️ **Ponto de atenção operacional**: o VPC Endpoint possui custo por hora e por GB processado — deve ser orçado antes do rollout. O cache agressivo nas bibliotecas internas é o principal mecanismo para controlar esse custo
- ⚠️ **Resiliência**: configurar o endpoint em múltiplas AZs para evitar ponto único de falha de rede

### **Resource Servers organizados por contexto de negócio**

⚠️ **Limite importante:** 25 Resource Servers (soft limit expansível para 300 via ticket AWS Support)

Considerando que temos **76 serviços apenas no Core** (e mais serviços em outros contextos), não é viável criar 1 Resource Server por microsserviço.

**Solução:** Organizar Resource Servers por **contexto de negócio**, onde cada um pode englobar até **100 scopes** (⚠️ limite fixo, sem possibilidade de reajuste de quota).

**Exemplos de organização por contexto:**

- **Resource Server: `fee-context`**
  - Scopes: `fee/all`, `fee/calcular`, `fee/aplicar`, `fee/estornar`
  - Atende: fee-api, billing-manager-api, etc.
- **Resource Server: `kyc-context`**
  - Scopes: `kyc/all`, `kyc/validacoes`, `kyc/compliance`
  - Atende: kyc-api, kyc-core-connector, kyc-person-complementation, etc.
- **Resource Server: `core-context`**
  - Scopes: `core/all`, `core/contas`, `core/transacoes`
  - Atende: core-api, core-read-api, etc.
- **Resource Server: `ledger-context`**
  - Scopes: `ledger/all`, `ledger/lancamentos`, `ledger/saldos`
  - Atende: ledger-api, etc.

**Governance dos Resource Servers**

⚠️ **Gap identificado (critical-thinking):** Quem é o dono de cada Resource Server? Sem um processo definido, escopos tendem a proliferar de forma desorganizada.

**Decisão:** Cada Resource Server deve ter um **time de domínio responsável** (owner). Novos escopos só são criados mediante Pull Request no repositório de IaC do Cognito, com aprovação do time dono. Isso garante rastreabilidade e evita que os 100 escopos por servidor sejam esgotados de forma não intencional.

### **App Clients (1 por serviço consumidor)**

- Cada serviço que **consome APIs** possui credenciais próprias (Client ID + Client Secret)
- 💰 **Importante:** Recentemente a AWS atualizou o modelo de precificação de M2M no Cognito, removendo o custo fixo de $6 por App Client. Agora o custo ocorre apenas na geração do token
- Credenciais armazenadas no **AWS Secrets Manager** com path padronizado: `/m2m/{ambiente}/{nome-do-servico}/client-secret`

### **Emissão de JWT**

- Tokens assinados com validade curta (TTL recomendado: **300 segundos / 5 minutos**)
- Contêm os escopos autorizados para o serviço no claim `scope`
- `ValidateAudience = false`: Cognito Access Tokens não possuem a claim `aud`

⚠️ **Gap de segurança identificado (devils-advocate):** Tokens já emitidos **não podem ser revogados imediatamente**. Se um App Client for comprometido, os tokens em circulação permanecem válidos até o `exp`. Com TTL de 5 minutos, a janela de exposição máxima é de 5 minutos após a rotação do segredo.

**Mitigação:** Rotacionar o client_secret imediatamente no Cognito (o que invalida novas emissões) e aguardar a janela de expiração. O runbook de resposta a incidentes deve documentar esse procedimento e o time de segurança deve estar ciente dessa janela.

### **JWKS Endpoint**

- Chaves públicas para validação de assinatura dos tokens
- Utilizado pelos serviços para validar tokens recebidos
- ⚠️ **Cache obrigatório**: o JWKS deve ser cacheado localmente com TTL de **1 hora**. Nunca buscar JWKS a cada requisição. Em caso de falha de validação de assinatura (evento de rotação de chaves do Cognito), recarregar automaticamente

### **Limites de taxa do Cognito**

⚠️ **Gap operacional identificado (devils-advocate):** O Cognito impõe por padrão **10 operações de token por segundo** por User Pool.

Com 76+ serviços, picos de cold-start (reinicialização simultânea de pods após deploy) podem estourar esse limite, causando falhas em cascata na autenticação M2M.

**Mitigações:**
- Cache in-process nas bibliotecas internas reduz drasticamente a frequência de emissão
- Para serviços com múltiplas réplicas: cache distribuído via ElastiCache Redis (evita thundering herd — múltiplas réplicas renovando o mesmo token simultaneamente)
- Renovação proativa 30 segundos antes da expiração com mutex/lock por instância
- **Solicitar elevação da quota via AWS Support antes do rollout geral**

### **Curva de aprendizado**

- Tendência a ser baixa, tendo em vista que já utilizamos Cognito na Aarin

---

## **2. Bibliotecas Internas (Abstração para Desenvolvedores)**

Criação de **bibliotecas M2M** que são pacotes internos que abstraem completamente a complexidade de obtenção, cache e renovação de tokens JWT, permitindo que desenvolvedores utilizem autenticação M2M de forma **transparente e sem código boilerplate**.

**Objetivo das bibliotecas:**

- Eliminar código repetitivo de autenticação em cada serviço
- Garantir implementação consistente e segura em toda a organização
- Facilitar a criação de novos serviços (plug-and-play)
- Centralizar evolução e correções de segurança
- Gerenciar cache de tokens automaticamente (reduzir chamadas ao Cognito)
- Renovação proativa de tokens antes da expiração

**Bibliotecas planejadas:**

**Para serviços consumidores (quem faz requisições):**

- **.NET (NuGet interno):** Integração com HttpClient, injeção automática de tokens
- **Node.js (NPM interno):** Integração com Axios/Fetch, interceptors automáticos
- **Lambda (Layer):** Para lambdas que precisam chamar outros serviços

**Para serviços consumidos (quem recebe requisições):**

- Middleware JWT padrão (.NET: `Microsoft.AspNetCore.Authentication.JwtBearer`, Node.js: `aws-jwt-verify`)

> ⚠️ **Gap identificado (critical-thinking):** O sucesso da solução depende criticamente da **adoção das bibliotecas**. Se equipes contornarem as bibliotecas e implementarem autenticação ad-hoc, os benefícios de padronização e segurança são perdidos.
>
> **Mitigação:** A etapa de CI/CD (descrita na seção de Prevenção de Compartilhamento de Credenciais) deve validar que a biblioteca oficial está sendo utilizada. O template `helm-aarin-base` deve tornar o uso das bibliotecas o caminho de menor resistência — mais fácil usar do que ignorar.

---

## **3. Validação JWT via Código em Cada Serviço**

Cada serviço que expõe APIs valida tokens usando middleware padrão da linguagem, exemplos simples:

**.NET:**

```csharp
// 1. Configuração de Autenticação (valida o token JWT)
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = "https://cognito-idp.us-east-1.amazonaws.com/us-east-1_XXX";
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidIssuer = "https://cognito-idp.us-east-1.amazonaws.com/us-east-1_XXX",
            ValidateAudience = false, // Cognito Access Tokens não possuem claim 'aud'
            ValidateLifetime = true
        };
    });

// 2. Configuração de Autorização (valida scopes)
builder.Services.AddAuthorization(options =>
{
    // Policy que verifica se o token tem o scope "core-context/all"
    options.AddPolicy("CoreAccess", policy =>
        policy.RequireClaim("scope", "core-context/all"));
    
    // Outras policies para scopes mais específicos (se necessário)
    options.AddPolicy("CreateAccount", policy =>
        policy.RequireClaim("scope", "core-context/criar-conta"));
});

// 3. Uso no Controller
[Authorize(Policy = "CoreAccess")] // ← Verifica scope "core-context/all"
[HttpGet("/api/accounts")]
public async Task<IActionResult> GetAccounts() { }

[Authorize(Policy = "CreateAccount")] // ← Verifica scope "core-context/criar-conta"
[HttpPost("/api/accounts")]
public async Task<IActionResult> CreateAccount() { }
```

**Node.js:**

```js
const { CognitoJwtVerifier } = require('aws-jwt-verify'); // biblioteca oficial AWS (substitui express-jwt, que está descontinuada)

const verifier = CognitoJwtVerifier.create({
  userPoolId: 'us-east-1_XXX',
  tokenUse: 'access',
  clientId: null, // Access Tokens M2M não vinculam clientId na validação
});

// Middleware de autenticação
app.use('/api/*', async (req, res, next) => {
  try {
    const token = req.headers.authorization?.split(' ')[1];
    req.auth = await verifier.verify(token);
    next();
  } catch {
    res.status(401).send('Não autorizado');
  }
});

// Middleware de verificação de escopo
app.use((req, res, next) => {
  const escopos = req.auth?.scope?.split(' ') || [];
  if (!escopos.includes('kyc-context/all')) {
    return res.status(403).send('Proibido: escopo insuficiente');
  }
  next();
});
```

> ⚠️ **Nota:** A biblioteca `express-jwt` mencionada na v2 está em modo manutenção. **Recomendamos `aws-jwt-verify`**, que é a biblioteca oficial da AWS para validação de tokens Cognito, com suporte ativo e cache nativo de JWKS.

### **Validação em Dupla Camada (Zero Trust)**

⚠️ **Gap de segurança identificado (se-system-architecture-reviewer):** Confiar apenas no API Gateway Authorizer cria um ponto cego — se uma requisição chegar ao serviço por outro caminho (ex: comunicação interna direta entre pods), ela não passaria pelo authorizer.

**Decisão:** Cada serviço exposto deve validar o JWT **independentemente**, mesmo quando o API Gateway já validou:

1. **API Gateway Authorizer**: primeira barreira (valida assinatura + escopos na borda)
2. **Middleware JWT no serviço**: segunda barreira (valida assinatura + `iss` + `exp` + escopo exigido)

Isso implementa o princípio de **Zero Trust**: nunca confiar, sempre verificar.

### **Tratamento de AWS Lambdas**

⚠️ **Importante:** JWT funciona apenas se a lambda estiver exposta para ser consumida por HTTP Request. Temos muitas lambdas que são consumidas por Invoke direto.

**Lambda exposta via API Gateway (HTTP):**

- **Validação no API Gateway** (sem código na Lambda)
- API Gateway possui Authorizer nativo que valida JWT do Cognito
- Lambda recebe requisição já autenticada/autorizada

**Lambda consumida via Invoke (não HTTP):**

- Validação via IAM Policies
- Controle de acesso baseado em roles IAM (quem pode invocar)
- Não utiliza JWT (não é necessário para comunicação interna AWS)

### **Prevenção de Compartilhamento de Credenciais**

**Problema:** Como garantir que o serviço X está utilizando suas próprias credenciais e não as de outro serviço?

**Temos algumas opções que tratam isso:**

**a) Template Helm atualizado - helm-aarin-base** ✅ **Decisão adotada**

- Injeção automática de credenciais via External Secrets Operator
- Path obrigatório: `/m2m/{environment}/{service-name}/client-secret`
- Cada pod recebe **apenas** as credenciais do seu próprio serviço — sem possibilidade de acesso às credenciais de outro serviço via Secrets Manager sem permissão IAM explícita

**b) Testes automatizados de arquitetura**

- Validar via testes de integração que o middleware de autorização está implementado (porém aqui teria que ser algo mais macro na pipe, pois o dev pode não fazer)

**c) Etapa no CI/CD** ✅ **Decisão adotada**

- Validar que a camada de autorização está implementada antes de permitir deploy
- Verificar presença de middleware JWT no código (script de validação a ser desenvolvido)
- Validar que credenciais corretas estão configuradas no Secrets Manager
- Verificar ausência de credenciais M2M hardcoded no código

### **Rotação de Segredos**

⚠️ **Gap operacional identificado (se-system-architecture-reviewer):** A v2 não definia como e quando os `client_secrets` seriam rotacionados.

**Decisão:** Implementar rotação automática via AWS Secrets Manager com ciclo de **90 dias**, usando Lambda de rotação que implementa o protocolo de 4 etapas:

1. `createSecret` — gera novo segredo
2. `setSecret` — atualiza o App Client no Cognito com o novo segredo
3. `testSecret` — valida que o novo segredo funciona (gera um token de teste)
4. `finishSecret` — promove o novo segredo como versão atual

Alarme CloudWatch configurado para falhas no ciclo de rotação.

### **Atenção: Scopes não possuem hierarquia**

⚠️ **Ponto de atenção crítico:** Scopes do Cognito **não possuem hierarquia**.

**Exemplo do problema:**

Se você tem um endpoint com os seguintes scopes:

- `servico/all`
- `servico/criar-conta`

O usuário/serviço precisará ter **ambos os escopos** para acessar o endpoint.

**Solução para Fase 1:**

Utilizar **scopes genéricos** por contexto para simplificar:

- `core-context/all`
- `kyc-context/all`
- `ledger-context/all`
- `fee-context/all`

Granularidade adicional pode ser implementada futuramente (Fase 2) se necessário, ou via validação adicional no código da aplicação.

### **Observabilidade e Auditoria**

⚠️ **Gap identificado (se-system-architecture-reviewer):** A v2 não definia como monitorar a saúde do sistema M2M em produção.

**Métricas obrigatórias no CloudWatch:**

- `TaxaEmissaoToken` por `client_id` — detecta serviços com cache mal configurado
- `FalhasValidacaoJWT` por serviço — detecta tokens expirados ou comprometidos
- `TentativasViolacaoEscopo` — detecta serviços tentando acessar recursos sem permissão
- `FalhasRotacaoSegredo` — alerta imediato se o ciclo de rotação falhar

**Dashboard CloudWatch** com painel de saúde M2M para o time de Plataforma/DevOps.

**Auditoria:** Todos os eventos de emissão de token são registrados no CloudTrail do Cognito com `client_id`, timestamp e escopos solicitados — habilitando rastreabilidade completa para fins de compliance.

### **Plano de Migração Keycloak → Cognito**

⚠️ **Gap identificado (critical-thinking):** A v2 propunha substituir o Keycloak, mas não definia como realizar essa transição sem interrupção dos serviços.

**Estratégia de migração em fases:**

1. **Fase 0 — Setup**: provisionar User Pool M2M, Resource Servers, VPC Endpoint e App Clients para serviços piloto (2-3 serviços do Core)
2. **Fase 1 — Piloto**: migrar serviços piloto para Cognito em paralelo ao Keycloak (dual-write, validação de ambos)
3. **Fase 2 — Rollout Core**: migrar todos os serviços do Core context, descomissionar tokens Keycloak para esse contexto
4. **Fase 3 — Rollout Geral**: expandir para demais contextos (kyc, fee, ledger, etc.)
5. **Fase 4 — Descomissionamento**: remover Keycloak M2M após todos os contextos migrados

> Estimativa para Fase 1 (core-to-core): **3–6 meses** conforme indicado na v2.

---

## **Trade-offs e alternativas consideradas**

### **Opção 1: Validação JWT no Código com AWS Cognito — Escolha para Fase 1**

**Arquitetura:**

- AWS Cognito para emissão de tokens M2M (removemos Keycloak)
- VPC Endpoint Privado (sem internet)
- Resource Servers por contexto de negócio
- **Bibliotecas internas** para obtenção de tokens (padronizamos implementação)
- **Validação JWT via código** em cada API (sem Istio)
  - .NET: `Microsoft.AspNetCore.Authentication.JwtBearer`
  - Node.js: `aws-jwt-verify`
  - Lambda: Validação no API Gateway ou IAM Policies

**Prós:**

- ✅ Muito mais simples de operar
- ✅ Desenvolvedores já conhecem (middleware JWT padrão)
- ✅ Sem overhead de sidecars
- ✅ Menor complexidade operacional
- ✅ VPC Endpoint reduz latência (sem internet)
- ✅ Implementação rápida (estimativa core-to-core: 3–6 meses)
- ✅ Ecossistema AWS nativo — sem dependência de SaaS externo
- ✅ Curva de aprendizado baixa (Cognito já é utilizado na Aarin)

**Contras:**

- ❌ Validação JWT vira responsabilidade de cada serviço (mitigado por bibliotecas + templates)
- ❌ Sem mTLS automático entre pods
- ❌ Sem canary/blue-green nativos
- ❌ Sem observabilidade de rede automática
- ❌ Janela de até 5 minutos de exposição após revogação de credencial comprometida

**Por que escolhemos esta opção:**

- Resolve **80% dos problemas** (token único, rastreabilidade, revogação granular) com **20% da complexidade**
- Permite validar modelo de scopes antes de investir em service mesh
- Facilita migração futura para Istio se necessário

### **Opção 2: Istio Service Mesh — REAVALIAÇÃO FUTURA (FASE 2)**

**Arquitetura:**

- Tudo da Opção 1 +
- Istio Service Mesh para validação na camada de rede

**Por que NÃO escolhemos agora:**

- Alta complexidade operacional
- Curva de aprendizado íngreme
- Requer equipe de SRE dedicada
- Não precisamos neste momento inicial

**Quando reavaliar Istio:**

✅ Crescimento significativo de microsserviços

✅ Requisitos de compliance obrigatórios (SOC 2, PCI-DSS com mTLS)

✅ Necessidade comprovada de canary deployments ou blue-green, circuit breakers

✅ Maturidade em Kubernetes consolidada

### **Opção 3: Manutenção do Keycloak/Gatekeeper — Rejeitada**

Manter o modelo atual não resolve nenhum dos problemas identificados: token compartilhado, ausência de rastreabilidade, violação do menor privilégio. Adicionalmente, o Keycloak opera por **exclusão de permissões**, o que torna o modelo propício a erros operacionais (esquecer de remover um escopo de um serviço novo). Rejeitado.

### **Opção 4: Auth0 / Okta para M2M — Rejeitada**

Provedores de identidade dedicados para M2M com modelo de escopos mais flexível e experiência de desenvolvedor superior. Rejeitados por introduzir dependência de SaaS de terceiros em uma organização AWS-nativa, custo adicional e esforço de integração sem benefício incremental para o contexto atual.

---

## **Conclusão**

As definições nesta ADR propõem uma evolução significativa na arquitetura de autenticação e autorização entre microsserviços da Aarin, substituindo o modelo atual de **token único compartilhado** por uma solução robusta baseada em **OAuth 2.0 Client Credentials** com **AWS Cognito** como provedor de identidade centralizado.

Em relação à v2, esta versão mantém a decisão arquitetural central e aprofunda a solução com:

- Ênfase nos detalhes de configuração e operação do AWS Cognito (TTL, limites de taxa, JWKS, governance de Resource Servers)
- Correção de gaps de segurança (Zero Trust em dupla camada, janela de revogação, rotação de segredos)
- Correção de gaps operacionais (plano de migração Keycloak→Cognito, observabilidade, alarmes)
- Correção de gaps de adoção (governança das bibliotecas internas, CI/CD enforcement)

## **Decisão Arquitetural Recomendada**

Após análise detalhada dos trade-offs, **recomendamos uma abordagem evolutiva em duas fases**:

### **Fase 1: Fundação Segura**

✅ **AWS Cognito M2M** como IdP centralizado (substituição do Keycloak)

✅ **Bibliotecas internas** (.NET, Node.js, Lambda) para abstração completa da autenticação

✅ **Validação JWT no código** de cada serviço (middleware padrão) com Zero Trust em dupla camada

✅ **Scopes globais** por contexto (`{contexto}/all`) para simplificar adoção inicial

✅ **Secrets Manager** para gerenciamento seguro de credenciais M2M com rotação automática de 90 dias

✅ **helm-aarin-base** + External Secrets Operator para prevenção de compartilhamento de credenciais

✅ **Observabilidade** via CloudWatch (métricas, alarmes, dashboard e auditoria via CloudTrail)

**Justificativa:** Esta fase resolve **80% dos problemas** identificados (token único, falta de rastreabilidade, ausência de revogação granular) com complexidade operacional mínima e permite que a Aarin:

- Elimine o ponto único de falha de segurança atual
- Ganhe rastreabilidade completa de comunicações entre serviços
- Estabeleça processo de gestão de identidades M2M
- Padronize autenticação em bibliotecas reutilizáveis
- Valide o modelo de scopes antes de investir em Service Mesh

### **Fase 2: Service Mesh com Istio — Avaliação futura (12 meses depois da Fase 1)**

**Reavaliar a necessidade de Istio** após consolidar a Fase 1, considerando:

**Gatilhos que justificam adoção do Istio:**

- ✅ Crescimento de microsserviços e necessidade de políticas uniformes
- ✅ **Requisitos de compliance** (SOC 2, PCI-DSS) exigindo mTLS obrigatório
- ✅ **Necessidade comprovada** de canary deployments, circuit breakers, rate limiting
- ✅ **Equipe de Platform Engineering** estabelecida (SREs dedicados)
- ✅ **Maturidade em Kubernetes** consolidada (troubleshooting avançado, GitOps maduro)

---

## Referências

- [RFC 6749 — The OAuth 2.0 Authorization Framework](https://datatracker.ietf.org/doc/html/rfc6749)
- [AWS Cognito — Client Credentials Grant](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-userpools-server-contract-reference.html)
- [AWS Cognito — Resource Servers and Scopes](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-define-resource-servers.html)
- [AWS Cognito — Service Quotas e Limites](https://docs.aws.amazon.com/cognito/latest/developerguide/limits.html)
- [AWS Secrets Manager — Rotação de Segredos](https://docs.aws.amazon.com/secretsmanager/latest/userguide/rotating-secrets.html)
- [aws-jwt-verify — Biblioteca Oficial AWS para Validação JWT](https://github.com/awslabs/aws-jwt-verify)
- [Istio — Security](https://istio.io/latest/docs/concepts/security/)
- [Istio — Authorization Policy](https://istio.io/latest/docs/reference/config/security/authorization-policy/)
- [Istio — Request Authentication](https://istio.io/latest/docs/reference/config/security/request_authentication/)
- [NIST — Zero Trust Architecture (SP 800-207)](https://csrc.nist.gov/publications/detail/sp/800-207/final)
- [IdentityModel — .NET OAuth 2.0 Client Library](https://github.com/IdentityModel/IdentityModel)
