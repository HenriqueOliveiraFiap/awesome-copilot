---
title: 'ADR-0001: M2M - Autenticação e Autorização entre Serviços Internos com AWS Cognito - V3'
status: 'Proposto'
date: '2026-04-07'
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
  - 'microsservicos'
  - 'segurança'
supersedes: 'ADR-v2: M2M - Autenticação e Autorização entre Serviços Internos'
superseded_by: ''
---

# ADR-0001: M2M — Autenticação e Autorização entre Serviços Internos com AWS Cognito — V3

## Status

**Proposto** — Supersede ADR-v2: M2M - Autenticação e Autorização entre Serviços Internos

---

## Contexto

### Histórico e Motivação

A ADR-v2 identificou o problema central: a comunicação entre microsserviços da Aarin utilizava um token compartilhado, onde um único token concedia acesso irrestrito a diversas APIs. O modelo do Gatekeeper/Keycloak operava por **exclusão de permissões** — um novo serviço recebia automaticamente todos os escopos disponíveis, exigindo remoção manual dos desnecessários.

Na prática, múltiplos serviços (pix-api, kyc-api, fee-api) compartilhavam as mesmas credenciais, tornando impossível identificar nos logs qual serviço específico executou cada ação.

A ADR-v2 propôs o AWS Cognito com OAuth 2.0 Client Credentials Grant como solução em **duas fases**: Fase 1 (validação JWT via código) e Fase 2 (Istio Service Mesh, avaliação futura). Esta ADR-v3 consolida as decisões da Fase 1 com maior precisão de implementação, incorpora aprendizados do processo de revisão e estabelece controles de governança que a v2 deixou em aberto.

> **Nota — pensamento crítico aplicado:** A decisão central da v2 (Cognito + Client Credentials) permanece correta. O que muda na v3 não é a direção estratégica, mas a precisão: governa os limites do Cognito que podem se tornar gargalos (25 Resource Servers, 100 scopes por servidor, 10 operações de token/s), define o ciclo de vida das bibliotecas internas, e endereça lacunas de governança identificadas durante a revisão — em especial, como impedir o compartilhamento de credenciais entre serviços em produção.

### O que Mudou da v2 para a v3

| Dimensão | ADR-v2 | ADR-v3 (este documento) |
|---|---|---|
| Foco | Proposta de solução + trade-offs | Decisões de implementação precisas |
| Bibliotecas internas | Mencionadas como planejadas | Especificadas: contratos, linguagens, padrões de cache |
| Governança de credenciais | Opções listadas sem decisão | Decisão tomada: helm-aarin-base + External Secrets Operator |
| Limites do Cognito | Citados como avisos | Documentados com estratégias de mitigação explícitas |
| Hierarquia de escopos | Advertência sobre ausência | Estratégia de Fase 1 (escopos globais) e critérios para Fase 2 definidos |
| Lambdas | Tratamento superficial | Dois padrões documentados: HTTP (API Gateway Authorizer) e Invoke (IAM Policies) |
| Observabilidade | Não abordada | Métricas CloudWatch, alarmes e dashboard especificados |
| Rotação de segredos | Não abordada | Ciclo de rotação via Secrets Manager + Lambda documentado |

---

## Decisão

**Implementar AWS Cognito com OAuth 2.0 Client Credentials Grant como provedor de identidade M2M centralizado para toda comunicação entre serviços internos da Aarin, substituindo definitivamente o Keycloak/Gatekeeper no contexto de autenticação máquina-a-máquina.**

A solução é sustentada por **3 pilares**:

1. **AWS Cognito** como autoridade central de identidade (emissão e gerenciamento de tokens)
2. **Bibliotecas internas** (.NET, Node.js, Lambda) para abstração completa do fluxo de autenticação nos serviços consumidores
3. **Validação JWT via código** em cada serviço exposto (middleware padrão por linguagem)

### Diagrama de Fluxo (Fase 1)

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Serviço A (Consumidor)                          │
│  1. Obtém client_secret do Secrets Manager (via biblioteca interna) │
│  2. POST /oauth2/token → Cognito (Client Credentials Grant)         │
│  3. Armazena JWT em cache (in-process, renovação proativa em T-30s) │
│  4. Adiciona Authorization: Bearer <JWT> na requisição              │
└──────────────────────────────┬──────────────────────────────────────┘
                               │ HTTPS + JWT (via VPC Endpoint Privado)
                               ▼
                 ┌─────────────────────────┐
                 │    API Gateway          │
                 │  Cognito Authorizer     │  ← valida assinatura e escopos
                 └────────────┬────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     Serviço B (Exposto)                             │
│  5. Middleware JWT valida token localmente via JWKS em cache        │
│  6. Verifica escopo necessário (ex: kyc-context/all)                │
│  7. Processa requisição                                             │
└─────────────────────────────────────────────────────────────────────┘
                              │
               ┌──────────────┴──────────────┐
               │        AWS Cognito          │
               │  User Pool M2M Dedicado     │
               │  VPC Endpoint Privado       │
               │  Resource Servers           │
               │  JWKS Endpoint              │
               └─────────────────────────────┘
```

---

## Pilar 1: AWS Cognito (Provedor de Identidade)

### User Pool M2M Dedicado

- Pool exclusivo para comunicação entre serviços (separado de usuários finais do BaaS e BaaS Admin)
- Domínio OAuth configurado para emissão de tokens via Client Credentials Grant

### VPC Endpoint Privado

- Comunicação interna sem tráfego de internet
- Reduz latência e aumenta a postura de segurança
- Custo adicional de VPC Endpoint deve ser orçado (ver NEG-5)

### Resource Servers Organizados por Contexto de Negócio

> ⚠️ **Limite crítico:** 25 Resource Servers por User Pool (soft limit, expansível para 300 via ticket AWS Support). Com **76 serviços no Core** e outros contextos, não é viável criar 1 Resource Server por microsserviço.

**Decisão:** Organizar Resource Servers por **contexto de negócio**, cada um com até **100 escopos** (limite fixo — sem possibilidade de ajuste de quota).

Exemplos de organização:

- **`fee-context`** → `fee/all`, `fee/calcular`, `fee/aplicar`, `fee/estornar` — atende: fee-api, billing-manager-api
- **`kyc-context`** → `kyc/all`, `kyc/validacoes`, `kyc/compliance` — atende: kyc-api, kyc-core-connector, kyc-person-complementation
- **`core-context`** → `core/all`, `core/contas`, `core/transacoes` — atende: core-api, core-read-api
- **`ledger-context`** → `ledger/all`, `ledger/lancamentos`, `ledger/saldos` — atende: ledger-api

> ⚠️ **Atenção — ausência de hierarquia em escopos:** Escopos do Cognito **não possuem hierarquia nativa**. Um endpoint protegido por `kyc-context/all` e `kyc-context/validacoes` exige que o App Client possua **ambos os escopos**. Para a **Fase 1**, utilizar **escopos globais por contexto** (`{contexto}/all`) simplifica a adoção e evita essa armadilha.

### App Clients (1 por Serviço Consumidor)

- Cada serviço que **consome APIs** possui suas próprias credenciais (Client ID + Client Secret)
- Credenciais armazenadas no **AWS Secrets Manager** com path obrigatório: `/m2m/{ambiente}/{nome-do-servico}/client-secret`
- **Sem custo fixo por App Client** (AWS atualizou precificação — cobrança ocorre apenas na geração do token)

### Tokens JWT

- Validade curta (TTL recomendado: 300 segundos / 5 minutos)
- Contêm os escopos autorizados para o serviço no claim `scope`
- **Validação de `aud` desabilitada**: Cognito Access Tokens não possuem a claim `aud`

---

## Pilar 2: Bibliotecas Internas

Criação de bibliotecas M2M que abstraem completamente a obtenção, cache e renovação de tokens JWT, permitindo que desenvolvedores utilizem autenticação M2M de forma **transparente e sem código boilerplate**.

### Para Serviços Consumidores (quem faz requisições)

- **.NET (NuGet interno):** Integração com `HttpClient`, injeção automática de token via `DelegatingHandler`
- **Node.js (NPM interno):** Integração com Axios/Fetch via interceptors automáticos
- **Lambda (Layer):** Para Lambdas que precisam chamar outros serviços

**Responsabilidades das bibliotecas:**

- **IMP-1 — Cache em dois níveis:** Cache in-process (singleton thread-safe) com expiração em `TTL - 30s`. Para serviços com múltiplas réplicas, cache distribuído via ElastiCache Redis para evitar thundering-herd de renovação de tokens simultâneas.
- **IMP-2 — Renovação proativa:** Disparada 30 segundos antes da expiração, com mutex/lock para evitar múltiplas renovações concorrentes na mesma instância.
- **IMP-3 — Cache do JWKS:** Chaves públicas do Cognito (`/.well-known/jwks.json`) cacheadas localmente com TTL de 1 hora. Recarregamento automático em falha de validação de assinatura (evento de rotação de chaves). **Nunca buscar JWKS por requisição.**

### Para Serviços Expostos (quem recebe requisições)

Middleware JWT padrão por linguagem — sem biblioteca proprietária adicional:

- **.NET:** `Microsoft.AspNetCore.Authentication.JwtBearer`
- **Node.js:** `aws-jwt-verify` (biblioteca oficial AWS — substituir `express-jwt` que está em manutenção)

---

## Pilar 3: Validação JWT via Código

### .NET

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

// 2. Configuração de Autorização (valida escopos)
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("AcessoCore", policy =>
        policy.RequireClaim("scope", "core-context/all"));

    options.AddPolicy("CriarConta", policy =>
        policy.RequireClaim("scope", "core-context/criar-conta"));
});

// 3. Uso no Controller
[Authorize(Policy = "AcessoCore")]
[HttpGet("/api/contas")]
public async Task<IActionResult> ListarContas() { }

[Authorize(Policy = "CriarConta")]
[HttpPost("/api/contas")]
public async Task<IActionResult> CriarConta() { }
```

### Node.js

```js
const { CognitoJwtVerifier } = require('aws-jwt-verify'); // biblioteca oficial AWS

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

### Tratamento de AWS Lambdas

| Tipo de Lambda | Mecanismo de Validação |
|---|---|
| Exposta via API Gateway (HTTP) | **Cognito Authorizer no API Gateway** — Lambda recebe requisição já autenticada/autorizada, sem código adicional |
| Consumida via Invoke direto (não HTTP) | **IAM Policies** — controle de acesso baseado em roles IAM (quem pode invocar). JWT não se aplica neste cenário |

---

## Consequências

### Positivas

- **POS-1 — Eliminação do Token Único Compartilhado:** Cada serviço possui identidade própria. Um token comprometido afeta apenas o serviço emissor, eliminando o ponto único de falha de segurança atual.

- **POS-2 — Rastreabilidade Completa:** Logs de acesso passam a conter o `client_id` do serviço originador, tornando possível identificar qual microsserviço acessou qual recurso em cada operação.

- **POS-3 — Menor Privilégio por Escopo:** App Clients recebem apenas os escopos estritamente necessários. A v3 define explicitamente escopos globais por contexto na Fase 1, com critérios claros para granularidade na Fase 2.

- **POS-4 — Abstração via Bibliotecas Internas:** Desenvolvedores não precisam entender OAuth 2.0 para consumir APIs internas. A biblioteca gerencia todo o ciclo de vida do token de forma transparente.

- **POS-5 — Baixa Curva de Aprendizado:** Cognito já é utilizado na Aarin para usuários finais; a equipe possui familiaridade com o serviço. Middleware JWT (.NET e Node.js) é padrão de mercado amplamente documentado.

- **POS-6 — Fundação para Fase 2 (Istio):** Ao concluir a Fase 1, a Aarin terá Cognito configurado, bibliotecas prontas, modelo de escopos validado e times treinados em OAuth 2.0 — reduzindo significativamente o esforço de uma eventual adoção de Istio.

### Negativas

- **NEG-1 — Limite de Taxa do Cognito (10 ops/s):** O Cognito impõe por padrão 10 operações de token por segundo por User Pool. Para frota com 76+ serviços, picos de cold-start (reinicialização simultânea de pods) podem estourar esse limite. **Mitigação:** cache em dois níveis nas bibliotecas internas (in-process + Redis) e renovação proativa reduzem drasticamente a frequência de emissão. Quota deve ser elevada via AWS Support antes do rollout geral.

- **NEG-2 — Responsabilidade de Validação Distribuída:** Sem Istio, cada serviço exposto é responsável por validar o token. Uma implementação incorreta em um serviço cria uma brecha localizada. **Mitigação:** bibliotecas de middleware padronizadas, validação por etapa de CI/CD (verificação de presença do middleware antes do deploy) e testes de arquitetura.

- **NEG-3 — Ausência de mTLS Nativo:** O Cognito com Client Credentials não oferece mTLS nativo. Comunicação M2M usa apenas JWT como credencial. **Mitigação:** tráfego interno via VPC Endpoint garante isolamento de rede. mTLS pode ser adicionado via Istio na Fase 2 se compliance exigir.

- **NEG-4 — Complexidade da Rotação de Segredos:** A Lambda de rotação deve atualizar atomicamente o Secrets Manager **e** o App Client do Cognito. Falha intermediária pode deixar as credenciais em estado inconsistente. **Mitigação:** Lambda de rotação idempotente implementando o ciclo `createSecret → setSecret → testSecret → finishSecret`, com alarme CloudWatch em caso de falha.

- **NEG-5 — Custo do VPC Endpoint:** VPC Endpoints têm custo por hora e por GB de dados processados. Para frota com alto volume de renovação de tokens, o custo é relevante. **Mitigação:** cache agressivo nas bibliotecas reduz o volume de chamadas ao endpoint. Custo deve ser orçado antes do rollout.

- **NEG-6 — Escopo Sem Hierarquia (Armadilha de Configuração):** Escopos do Cognito não possuem herança. Se um endpoint exige `kyc-context/all` e `kyc-context/validacoes`, o App Client precisa de ambos explicitamente. **Mitigação na Fase 1:** utilizar apenas escopos globais por contexto (`{contexto}/all`). Granularidade adicional somente via validação no código da aplicação.

- **NEG-7 — Vendor Lock-in AWS:** A solução é integralmente dependente do ecossistema AWS (Cognito, Secrets Manager, API Gateway, CloudWatch). Migração para multi-cloud ou provedor de identidade agnóstico exigiria re-arquitetura do plano de identidade M2M. Risco aceitável dado o perfil AWS-nativo da Aarin.

---

## Alternativas Consideradas

- **ALT-1 — Manutenção do Keycloak/Gatekeeper:** Modelo atual. Operação por exclusão de permissões, token compartilhado, sem rastreabilidade. Rejeitado por violar os princípios de menor privilégio e auditabilidade exigidos pela maturidade de segurança da Aarin.

- **ALT-2 — Istio Service Mesh (Fase 1):** Istio permite validação de identidade na camada de rede com mTLS automático, canary deployments e observabilidade nativa. Rejeitado para a Fase 1 por alta complexidade operacional, curva de aprendizado íngreme e necessidade de equipe SRE dedicada que a Aarin não possui neste momento. Reavaliação prevista após consolidação da Fase 1 (12 meses), quando os gatilhos abaixo forem atingidos: crescimento significativo de microsserviços, requisitos de compliance obrigatórios (SOC 2, PCI-DSS), necessidade comprovada de circuit breakers/canary e maturidade em Kubernetes consolidada.

- **ALT-3 — Auth0 / Okta para M2M:** Provedores de identidade dedicados para M2M com modelo de escopos mais flexível, rotação nativa de segredos e melhor experiência de desenvolvedor. Rejeitados por introduzir dependência de SaaS de terceiros em uma organização AWS-nativa, custo adicional e esforço de integração sem benefício incremental para o contexto atual.

- **ALT-4 — IAM Roles (SigV4) para Comunicação Interna AWS:** Para serviços hospedados integralmente na AWS, IAM Roles com SigV4 eliminam a necessidade de emissão de tokens. Rejeitado como mecanismo primário pois nem todos os serviços possuem IAM Role (workloads containerizados), e SigV4 não funciona para serviços fora do boundary AWS. **Recomendado como padrão complementar** para comunicação Lambda-to-Lambda dentro da mesma conta.

- **ALT-5 — API Keys Estáticas (sem OAuth):** Simplicidade máxima, mas sem escopos, sem rotação automática, sem rastreabilidade e sem possibilidade de revogação granular. Rejeitado por não resolver nenhum dos problemas identificados na v2.

---

## Notas de Implementação

- **IMP-1 — Configuração do User Pool M2M:** Criar User Pool dedicado exclusivamente para M2M (não reutilizar o pool de usuários finais). Habilitar o domínio OAuth. Registrar Resource Servers por contexto de negócio com seus escopos antes de provisionar App Clients.

- **IMP-2 — Provisionamento de App Clients:** Criar 1 App Client por serviço consumidor. Desabilitar todos os grant types exceto `client_credentials`. Atribuir apenas os escopos mínimos necessários. Armazenar `client_id` e `client_secret` no Secrets Manager no path padrão: `/m2m/{ambiente}/{nome-do-servico}/client-secret`.

- **IMP-3 — Governança via helm-aarin-base:** O template Helm `helm-aarin-base` deve ser atualizado para injetar automaticamente as credenciais M2M via External Secrets Operator a partir do path padronizado do Secrets Manager. Isso garante que cada pod utilize **suas próprias credenciais** e não as de outro serviço.

- **IMP-4 — Validação no CI/CD:** Adicionar etapa no pipeline de deploy que verifica: (a) presença do middleware JWT no código do serviço exposto; (b) ausência de credenciais M2M hardcoded; (c) credenciais corretas configuradas no Secrets Manager para o ambiente de destino.

- **IMP-5 — Lambda de Rotação de Segredos:** Implementar o ciclo de rotação do Secrets Manager em 4 etapas: `createSecret` (gera novo segredo) → `setSecret` (atualiza App Client no Cognito) → `testSecret` (valida que novo segredo funciona) → `finishSecret` (promove novo segredo como atual). Ciclo de rotação: **90 dias**. Alarme CloudWatch em qualquer falha do ciclo.

- **IMP-6 — Configuração do API Gateway Authorizer:** Configurar Cognito Authorizer com ARN do User Pool M2M. Especificar os escopos obrigatórios por rota. Definir TTL do resultado do authorizer como **0** para fluxos M2M (tokens são de curta duração e cacheados no cliente — cache do authorizer pode causar verificação de escopos desatualizada).

- **IMP-7 — Observabilidade:** Publicar métricas customizadas no CloudWatch: `TaxaEmissaoToken` (por client_id), `FalhasValidacaoJWT` (por serviço), `TentativasViolacaoEscopo`. Criar Dashboard CloudWatch com saúde M2M. Alarmes em: `FalhasValidacaoJWT > limiar`, `FalhaRotacaoSegredo > 0`, `TaxaEmissaoToken > 80% da quota do Cognito`.

- **IMP-8 — Zero Trust em Camada Dupla:** Cada serviço exposto deve validar o JWT independentemente, mesmo quando API Gateway já validou. Validar: assinatura (JWKS), `iss` (URL do emissor Cognito), `exp` (não expirado) e escopo exigido. Não confiar apenas na decisão do API Gateway.

- **IMP-9 — Runbook de Resposta a Incidentes:** Documentar o processo de revogação de App Client comprometido: rotacionar segredo imediatamente no Cognito + Secrets Manager → alertar serviços downstream → aguardar janela de expiração do token (até 300s, tokens emitidos permanecem válidos até o `exp`). Planejar estratégia para essa janela no runbook de incidentes.

---

## Referências

- **REF-1** — [RFC 6749 — The OAuth 2.0 Authorization Framework](https://datatracker.ietf.org/doc/html/rfc6749)
- **REF-2** — [AWS Cognito — Client Credentials Grant](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-userpools-server-contract-reference.html)
- **REF-3** — [AWS Cognito — Resource Servers e Escopos](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-define-resource-servers.html)
- **REF-4** — [AWS Secrets Manager — Rotação de Segredos](https://docs.aws.amazon.com/secretsmanager/latest/userguide/rotating-secrets.html)
- **REF-5** — [AWS API Gateway — Cognito Authorizer](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-integrate-with-cognito.html)
- **REF-6** — [AWS Cognito — Service Quotas](https://docs.aws.amazon.com/cognito/latest/developerguide/limits.html)
- **REF-7** — [aws-jwt-verify — Biblioteca Oficial AWS para Validação JWT](https://github.com/awslabs/aws-jwt-verify)
- **REF-8** — [IdentityModel — .NET OAuth 2.0 Client Library](https://github.com/IdentityModel/IdentityModel)
- **REF-9** — [NIST SP 800-207 — Zero Trust Architecture](https://csrc.nist.gov/publications/detail/sp/800-207/final)
- **REF-10** — [Istio — Security](https://istio.io/latest/docs/concepts/security/)
- **REF-11** — [Istio — Authorization Policy](https://istio.io/latest/docs/reference/config/security/authorization-policy/)
- **REF-12** — ADR-v2: M2M - Autenticação e Autorização entre Serviços Internos *(documento interno supersedido por este)*
