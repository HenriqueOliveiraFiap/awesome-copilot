| **Campo** | **Valor** |
| --- | --- |
| **Status** | Em revisão |
| **Data** | 2026-04-08 |
| **Autores** | Henrique Oliveira |
| **Revisores** | Tecnologia, Plataforma, DevOps e Segurança |

---

**Contexto e problema**

Atualmente, a comunicação entre os microsserviços da Aarin utiliza um token que pode ser compartilhado entre serviços, onde um único token tem capacidade para acessar diversas APIs.

O fluxo de criação de usuários no Gatekeeper/Keycloak funciona por exclusão de permissões:

- Um novo serviço criado recebe automaticamente todos os escopos disponíveis (todas as roles)
- Para restringir acesso, seria necessário **remover manualmente** os escopos desnecessários

Na prática operacional, múltiplos serviços utilizam **as mesmas credenciais** para autenticação:

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

**Solução**

Adotar **AWS Cognito** como IdP (Identity Provider) dedicado para comunicação M2M, implementando o fluxo **OAuth 2.0 Client Credentials Grant** para autenticação entre serviços internos.

A solução utiliza controle de acesso baseado em **escopos**, onde cada endpoint é protegido por um ou mais escopos, e cada serviço consumidor possui apenas os escopos estritamente necessários para suas operações.

**A validação de tokens JWT será feita via código** em cada serviço (.NET, Node.js) utilizando middleware padrão, com suporte de **bibliotecas internas** que abstraem a complexidade de obtenção, cache e renovação de tokens.

**Decisão: Por que AWS Cognito?**

Durante a análise de alternativas, foram considerados outros provedores de identidade (Auth0, Okta, manutenção do Keycloak). A escolha pelo AWS Cognito se deu pelos seguintes motivos:

- **Já utilizado na Aarin**: o Cognito é o IdP atual para usuários finais (BaaS, BaaS Admin), o que significa familiaridade da equipe, infraestrutura já provisionada e curva de aprendizado reduzida
- **Ecossistema AWS nativo**: integração direta com Secrets Manager, API Gateway Authorizer, CloudTrail e IAM — sem dependência de SaaS externo de terceiros
- **Precificação M2M atualizada**: a AWS removeu o custo fixo de $6 por App Client; o custo ocorre apenas na geração de tokens, o que favorece o modelo de 1 App Client por serviço
- **Suporte a Client Credentials Grant**: fluxo OAuth 2.0 nativo para M2M, sem necessidade de adaptações

**Decisão: Validação no código (Fase 1)**

**Para a Fase 1**, seguiremos com **validação JWT via código** (middleware padrão), que resolve 80% dos problemas com 20% da complexidade. Futuras evoluções como Service Mesh (Istio) são descritas na seção de **Fase 2** ao final desta ADR.

A solução é composta por **3 pilares principais**:

---

**1. AWS Cognito (Provedor de Identidade)**

O Cognito atua como autoridade central de identidade para todos os serviços internos:

**User Pool M2M dedicado**

- Exclusivo para comunicação entre serviços (separado de usuários finais como BaaS ou BaaS Admin)
- Domínio OAuth habilitado para emissão de tokens via Client Credentials Grant
- **Isolamento total**: nenhum usuário humano tem acesso a este pool

**Resource Servers organizados por contexto de negócio**

⚠️ **Limite importante:** 25 Resource Servers (soft limit expansível para 300 via ticket AWS Support)

Considerando que temos **76 serviços apenas no Core** (e mais serviços em outros contextos), não é viável criar 1 Resource Server por microsserviço.

**Solução:** Organizar Resource Servers por **contexto de negócio**, onde cada um pode englobar até **100 scopes** (⚠️ limite fixo, sem possibilidade de reajuste de quota).

**Exemplos de organização por contexto:**

- **Resource Server: `fee-context`**
    - Scopes: `fee/all`, `fee/calcular`, `fee/aplicar`, `fee/estornar`
    - Atende: fee-api, billing-manager-api, etc.
- **Resource Server: `kyc-context`**
    - Scopes: `kyc/all`, `kyc/validacoes`, `kyc/compliance`
    - Atende: kyc-api, kyc-core-connector, kyc-person-complementation, etc.
- **Resource Server: `core-context`**
    - Scopes: `core/all`, `core/contas`, `core/transacoes`
    - Atende: core-api, core-read-api, etc.
- **Resource Server: `ledger-context`**
    - Scopes: `ledger/all`, `ledger/lancamentos`, `ledger/saldos`
    - Atende: ledger-api, etc.

**Governance dos Resource Servers**

⚠️ **Gap identificado:** Quem é o dono de cada Resource Server? Sem um processo definido, escopos tendem a proliferar de forma desorganizada.

**Decisão:** Cada Resource Server deve ter um **time de domínio responsável** (owner). Novos escopos só são criados mediante Pull Request no repositório de IaC do Cognito, com aprovação do time dono. Isso garante rastreabilidade e evita que os 100 escopos por servidor sejam esgotados de forma não intencional.

**App Clients (1 por serviço consumidor)**

- Cada serviço que **consome APIs** possui credenciais próprias (Client ID + Client Secret)
- 💰 **Importante:** Recentemente a AWS atualizou o modelo de precificação de M2M no Cognito, removendo o custo fixo de $6 por App Client. Agora o custo ocorre apenas na geração do token, com precificação escalonada ([AWS Cognito Pricing](https://aws.amazon.com/cognito/pricing/)):
    
    
    | **Tier** | **Faixa (tokens/mês)** | **Preço por 1.000 tokens** |
    | --- | --- | --- |
    | 1 | 1 – 250.000 | **USD 2,25** |
    | 2 | 250.001 – 5.000.000 | USD 1,50 |
    | 3 | 5.000.001+ | USD 1,125 |
    
    **Estimativa de custo para a Aarin (~300 serviços, 2 pods cada, cache de 1h):**
    
    Fórmula: `(tokens ÷ 1.000) × preço_do_tier`
    
    | **Cenário** | **Tokens/mês** | **Cálculo por tier** | **Custo estimado/mês** | **Custo/serviço/mês** (2 pods) |
    | --- | --- | --- | --- | --- |
    | **Otimista** — cache distribuído com mutex: 1 token/serviço/hora | 300 × 24 × 30 = **216.000** | 216 × $2,25 (Tier 1) | **~$486** | 720 tokens × $2,25/1.000 = **~$1,62** |
    | **Realista** — cache in-process por pod (sem mutex): 2 tokens/serviço/hora | 600 × 24 × 30 = **432.000** | 250 × $2,25 + 182 × $1,50 (Tier 1+2) | **~$835** | 1.440 tokens × $2,25/1.000 = **~$3,24** |
    | **Conservador** — inclui cold-starts de deploys (10 rollouts/mês × 600 pods) | ~438.000 | 250 × $2,25 + 188 × $1,50 (Tier 1+2) | **~$845** | ~1.460 tokens × $2,25/1.000 = **~$3,29** |
    
    > 💡 **Custo por serviço:** Considerando o mínimo de 2 pods por serviço com cache in-process (cenário Realista), cada serviço emite 2 tokens/hora × 24h × 30 dias = **1.440 tokens/mês**, custando aproximadamente **$3,24/serviço/mês**. Com cache distribuído e mutex (cenário Otimista), os 2 pods compartilham 1 único token cacheado por hora, reduzindo para 720 tokens/mês e **~$1,62/serviço/mês**.
    >
    > 💡 **Conclusão:** O cenário Otimista (com cache distribuído e mutex, evitando requisições duplicadas entre pods) mantém o volume abaixo de 250.000/mês e fica inteiramente no **Tier 1 (~$486/mês)**. Os cenários Realista e Conservador ultrapassam esse limite e entram no Tier 2, custando **~$835–$845/mês**. O principal mecanismo de controle de custo é o cache de 1 hora nas bibliotecas internas — que também protege contra os limites de taxa do Cognito (10 ops/s por User Pool). Considerar cache distribuído com mutex reduz o custo em ~42%.
    > 
- Credenciais armazenadas no **AWS Secrets Manager** com path padronizado: `/m2m/{ambiente}/{nome-do-servico}/client-secret`

**Emissão de JWT**

- Tokens assinados com validade de **1 hora** (TTL padrão do Cognito para Access Tokens, compatível com o cache das bibliotecas internas)
- Contêm os escopos autorizados para o serviço no claim `scope`
- `ValidateAudience = false`: Cognito Access Tokens não possuem a claim `aud`

**JWKS Endpoint**

- Chaves públicas para validação de assinatura dos tokens
- Utilizado pelos serviços para validar tokens recebidos
- ⚠️ **Cache obrigatório**: o JWKS deve ser cacheado localmente com TTL de **1 hora**. Nunca buscar JWKS a cada requisição. Em caso de falha de validação de assinatura (evento de rotação de chaves do Cognito), recarregar automaticamente

**Limites de taxa do Cognito**

⚠️ **Gap operacional:** O Cognito impõe por padrão **10 operações de token por segundo** por User Pool.

Com cache de 1 hora nas bibliotecas internas, a frequência de chamadas ao Cognito é drasticamente reduzida (cerca de 1 emissão por serviço por hora). Ainda assim, picos de cold-start (reinicialização simultânea de pods após deploy) podem estourar o limite.

**Mitigações:**

- Cache in-process com TTL de 1 hora nas bibliotecas internas (principal mecanismo)
- Renovação proativa 60 segundos antes da expiração com mutex/lock por instância (evita múltiplas réplicas do mesmo serviço renovando o token ao mesmo tempo)
- **Solicitar elevação da quota via AWS Support antes do rollout geral**

**Curva de aprendizado**

- Tendência a ser baixa, tendo em vista que já utilizamos Cognito na Aarin

---

**2. Bibliotecas Internas (Abstração para Desenvolvedores)**

Criação de **bibliotecas M2M** que são pacotes internos que abstraem completamente a complexidade de obtenção, cache e renovação de tokens JWT, permitindo que desenvolvedores utilizem autenticação M2M de forma **transparente e sem código boilerplate**.

**Objetivo das bibliotecas:**

- Eliminar código repetitivo de autenticação em cada serviço
- Garantir implementação consistente e segura em toda a organização
- Facilitar a criação de novos serviços (plug-and-play)
- Centralizar evolução e correções de segurança
- Gerenciar cache de tokens automaticamente (reduzir chamadas ao Cognito)
- Renovação proativa de tokens antes da expiração

**Bibliotecas planejadas:**

**Para serviços consumidores (quem faz requisições):**

- **.NET (NuGet interno):** Integração com HttpClient, injeção automática de tokens
- **Node.js (NPM interno):** Integração com Axios/Fetch, interceptors automáticos
- **Lambda (Layer):** Para lambdas que precisam chamar outros serviços

**Para serviços consumidos (quem recebe requisições):**

- Middleware JWT padrão (.NET: `Microsoft.AspNetCore.Authentication.JwtBearer`, Node.js: `aws-jwt-verify`)

> ⚠️ **Gap:** O sucesso da solução depende criticamente da **adoção das bibliotecas**. Se equipes contornarem as bibliotecas e implementarem autenticação ad-hoc, os benefícios de padronização e segurança são perdidos.
> 
> 
> **Mitigação:** A etapa de CI/CD (descrita na seção de Prevenção de Compartilhamento de Credenciais) deve validar que a biblioteca oficial está sendo utilizada. O template `helm-aarin-base` deve tornar o uso das bibliotecas o caminho de menor resistência — mais fácil usar do que ignorar.
> 

---

**2.1 Estratégia de Cache e Renovação de Tokens**

Um dos pilares mais importantes das bibliotecas internas é o **gerenciamento de ciclo de vida do token**: obter um token somente quando necessário, mantê-lo válido durante sua vigência e renová-lo **de forma proativa**, sem jamais deixar uma requisição falhar por token expirado. Esta seção descreve as duas abordagens disponíveis e os critérios para escolha entre elas.

**Princípio: Renovação Proativa (Pre-emptive Refresh)**

Independentemente da abordagem de cache escolhida, as bibliotecas devem seguir o seguinte contrato de ciclo de vida:

![image.png](attachment:df869af7-4aa9-4357-82c9-6fdb74968cf4:image.png)

**Regras das bibliotecas:**

1. **Renovar 60 segundos antes da expiração** — garante que o token esteja sempre fresco quando injetado em uma requisição HTTP, sem buracos de autenticação entre threads concorrentes
2. **Mutex por instância** — evitar que múltiplas threads do mesmo pod disparem chamadas simultâneas ao Cognito para o mesmo `client_id` (thundering herd local)
3. **Fallback tolerante** — se a renovação proativa falhar (Cognito indisponível), a biblioteca deve ainda tentar usar o token em cache enquanto ele não tiver expirado, e só lançar exceção quando não houver token válido algum
4. **Cache sensível ao `exp`** — o TTL de cache deve ser calculado dinamicamente a partir do claim `exp` do JWT retornado, não de um valor fixo hardcoded. Isso garante compatibilidade se o Cognito retornar um TTL diferente de 3600s no futuro

**O que é um Mutex e por que ele é necessário aqui?**

**Mutex** (do inglês *Mutual Exclusion*) é um mecanismo de sincronização que garante que apenas **uma thread por vez** execute um bloco de código crítico. Todas as outras threads que tentam entrar naquele bloco ficam em fila e aguardam a thread atual terminar antes de prosseguir.

**Por que precisamos de mutex neste contexto?**

Servidores de produção processam múltiplas requisições HTTP em paralelo — cada uma em sua própria thread. Se 20 requisições chegarem ao mesmo tempo e o token estiver expirado (ou ainda não emitido), **todas as 20 threads verão o cache vazio e tentarão emitir um novo token ao mesmo tempo**. Isso causaria:

- 20 chamadas simultâneas ao Cognito para o mesmo `client_id`
- Risco de exceder o limite de 10 ops/s do User Pool
- Emissão de 20 tokens desnecessários (apenas 1 será usado)
- Custo desnecessário no modelo de precificação por token

O mutex resolve isso com o padrão **double-check locking**:

1. Thread A adquire o lock (as demais ficam aguardando)
2. Thread A verifica o cache novamente (double-check) — está vazio → emite token → salva no cache → libera o lock
3. Thread B adquire o lock → verifica o cache → **token já está lá** → retorna sem chamar o Cognito
4. Threads C, D, ... fazem o mesmo que B

Resultado: **apenas 1 chamada ao Cognito**, independentemente de quantas threads chegaram ao mesmo tempo.

Em .NET, o mutex assíncrono é implementado com `SemaphoreSlim(1, 1)` — um semáforo configurado para permitir exatamente 1 entrada simultânea, com o método `WaitAsync()` que suspende a coroutine sem bloquear a thread do pool (essencial em código `async/await`).

---

**Abordagem 1 — Cache Local (In-Process / IMemoryCache)**

Cada pod mantém seu próprio cache em memória. É a abordagem padrão para a **Fase 1**.

**Funcionamento:**

![image.png](attachment:0b67046d-b3a8-4de5-b507-100a5c7b4091:image.png)

**Por que múltiplos tokens simultâneos são aceitos:**

O Cognito **não invalida tokens anteriores** quando um novo é emitido para o mesmo `client_id`. Dois tokens emitidos em momentos diferentes para o mesmo App Client são igualmente válidos até seu respectivo `exp`. Isso é intencional no protocolo — o receptor valida apenas assinatura, `iss` e `exp`, independentemente de quantos tokens existem em circulação.

**Implementação .NET:**

```csharp
public class CognitoTokenService : ICognitoTokenService
{
    private readonly IMemoryCache _cache;
    private readonly SemaphoreSlim _lock = new(1, 1);
    private readonly CognitoM2MOptions _options;

    public async Task<string> GetTokenAsync(CancellationToken ct = default)
    {
        var cacheKey = $"m2m_token_{_options.ClientId}";

        if (_cache.TryGetValue(cacheKey, out string cachedToken))
            return cachedToken;

        await _lock.WaitAsync(ct);
        try
        {
            // Double-check após adquirir o lock
            if (_cache.TryGetValue(cacheKey, out cachedToken))
                return cachedToken;

            var response = await _cognitoClient.IssueTokenAsync(_options);
            var expiresIn = response.ExpiresIn; // segundos até expiração (ex: 3600)
            var renewBefore = TimeSpan.FromSeconds(expiresIn - 60); // renovar 60s antes

            _cache.Set(cacheKey, response.AccessToken, renewBefore);
            return response.AccessToken;
        }
        finally
        {
            _lock.Release();
        }
    }
}
```

**Prós:**

- ✅ Zero dependência de infraestrutura externa (sem Redis, sem ElastiCache)
- ✅ Latência de cache próxima de zero (memória local do processo)
- ✅ Sem custo adicional de infraestrutura
- ✅ Sem complexidade operacional adicional
- ✅ Totalmente suficiente para a grande maioria dos cenários com cache de 1h

**Contras:**

- ❌ Cada pod emite seu próprio token ao iniciar (cold-start) — impacto na taxa do Cognito durante scale-out rápido
- ❌ Sem coordenação entre réplicas — `N` pods = até `N` tokens em circulação
- ❌ Cold-start em batch (ex: deploy que reinicia 50 pods simultaneamente) pode saturar o limite de 10 ops/s do Cognito
- ❌ Cache perdido ao reiniciar o pod (próximo start emite novo token)

**Quando é suficiente:**

| **Fator** | **Threshold seguro para cache local** |
| --- | --- |
| Número de pods por serviço | ≤ 10–15 réplicas |
| Frequência de scale-out (HPA) | Escalonamento gradual (não instantâneo de 0→N) |
| Número total de serviços × pods | ≤ ~300 pods simultâneos (para ficar abaixo de 10 ops/s médio) |
| Frequência de deploys | ≤ 10 deploys/dia com rollout gradual |

---

**Abordagem 2 — Cache Distribuído (Redis / ElastiCache)**

Todas as réplicas de um mesmo serviço compartilham um único cache externo. É a abordagem para cenários de **alta escala ou HPA agressivo**.

**Funcionamento:**

![image.png](attachment:a61ed31e-3569-4a96-9f18-24ca93088494:image.png)

**Implementação .NET (com `IDistributedCache`):**

```csharp
public class CognitoTokenService : ICognitoTokenService
{
    private readonly IDistributedCache _distributedCache;
    private readonly IConnectionMultiplexer _redis;  // para mutex distribuído
    private readonly CognitoM2MOptions _options;

    public async Task<string> GetTokenAsync(CancellationToken ct = default)
    {
        var cacheKey = $"m2m_token_{_options.ClientId}";
        var lockKey = $"m2m_lock_{_options.ClientId}";

        var cached = await _distributedCache.GetStringAsync(cacheKey, ct);
        if (cached is not null) return cached;

        // Mutex distribuído — apenas 1 pod emite o token, os demais aguardam
        await using var redisLock = await _redis.AcquireLockAsync(lockKey, timeout: 5s);
        {
            cached = await _distributedCache.GetStringAsync(cacheKey, ct);
            if (cached is not null) return cached;

            var response = await _cognitoClient.IssueTokenAsync(_options);
            var ttl = TimeSpan.FromSeconds(response.ExpiresIn - 60);

            await _distributedCache.SetStringAsync(cacheKey, response.AccessToken,
                new DistributedCacheEntryOptions { AbsoluteExpirationRelativeToNow = ttl }, ct);

            return response.AccessToken;
        }
    }
}
```

> 💡 A abstração `IDistributedCache` do .NET permite trocar a implementação entre Redis ou Memcached sem alterar o código da biblioteca, apenas a configuração do DI muda.
> 

**Prós:**

- ✅ 1 token por serviço (independentemente do número de pods) — custo mínimo com Cognito
- ✅ Scale-out instantâneo (100 novos pods) não gera 100 chamadas ao Cognito — todos leem do Redis
- ✅ Resiliente a cold-starts massivos de deploy
- ✅ Elimina risco de saturar o limite de 10 ops/s do Cognito em cenários de HPA agressivo
- ✅ Cache compartilhado sobrevive ao reinício de pods individuais

**Contras:**

- ❌ Dependência de infraestrutura Redis/ElastiCache (custo, disponibilidade, latência de rede)
- ❌ Complexidade operacional maior (gestão de ElastiCache, HA, failover)
- ❌ Latência de leitura de cache levemente maior (rede vs. memória local)
- ❌ Se o Redis ficar indisponível, todos os pods perdem o cache simultaneamente (single point of failure de cache)
- ❌ Necessidade de mutex distribuído (mais complexidade de implementação)

**Quando justifica adoção:**

| **Fator** | **Threshold que justifica Redis** |
| --- | --- |
| Pods por serviço | > 15 réplicas estáveis |
| HPA | Scale-out de ≥ 20 pods em < 60s frequentemente |
| Deploys simultâneos | > 10 serviços fazendo rollout em paralelo |
| Orçamento Cognito | Necessidade de ficar abaixo do Tier 1 (< 250k tokens/mês) |
| Compliance | Rastreabilidade de qual token específico está em uso em cada momento |

---

**Comparativo das Abordagens**

| **Dimensão** | **Cache Local (IMemoryCache)** | **Cache Distribuído (Redis)** |
| --- | --- | --- |
| **Complexidade** | Baixa | Alta |
| **Custo de infra** | Zero | Redis/ElastiCache (~$50–200/mês para HA) |
| **Tokens simultâneos** | 1 por pod | 1 por serviço |
| **Resiliência a scale-out** | Moderada | Alta |
| **Latência de cache hit** | ~0ms (memória) | ~1–5ms (rede local) |
| **Impacto em HPA agressivo** | Alto (N novos pods = N novas emissões) | Mínimo (pods leem do Redis) |
| **Implementação** | ~30 linhas, sem dependências externas | ~60 linhas + dependência Redis + mutex distribuído |
| **Fase recomendada** | **Fase 1** | Fase 2 (critérios acima) |

---

> ⚠️ **Disclaimer — Estratégia de Evolução Gradual**
> 
> 
> **Para a Fase 1, adotaremos Cache Local (IMemoryCache) como estratégia padrão das bibliotecas internas.** O cache local resolve o problema de custo e rate limit do Cognito para a grande maioria dos cenários da Aarin no momento atual, sem introduzir dependência de infraestrutura adicional.
> 
> **A migração para Cache Distribuído (Redis) será considerada como evolução em uma fase futura**, quando um ou mais dos seguintes gatilhos forem observados em produção:
> 
> - Serviços com HPA configurado para escalar agressivamente (ex: 0→50 pods em burst)
> - Alertas de `m2m.token.emissao` no Datadog indicando pico de emissões próximo ao limite de 10 ops/s do Cognito
> - Volume mensal de tokens aproximando-se de 250.000 (fronteira entre Tier 1 e Tier 2 de custo)
> - Rollouts simultâneos frequentes de grande número de serviços
> 
> As bibliotecas internas serão projetadas desde o início com **inversão de dependência** (`ITokenCache` como abstração), de modo que a troca de cache local para distribuído seja uma mudança de configuração de DI, sem necessidade de alterar o código dos serviços consumidores.
> 

---

**3. Validação JWT via Código em Cada Serviço**

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

```jsx
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

**Validação em Dupla Camada (Zero Trust)**

⚠️ **Gap de segurança:** Confiar apenas no API Gateway Authorizer cria um ponto cego — se uma requisição chegar ao serviço por outro caminho (ex: comunicação interna direta entre pods), ela não passaria pelo authorizer.

**Decisão:** Cada serviço exposto deve validar o JWT **independentemente**, mesmo quando o API Gateway já validou:

1. **API Gateway Authorizer**: primeira barreira (valida assinatura + escopos na borda)
2. **Middleware JWT no serviço**: segunda barreira (valida assinatura + `iss` + `exp` + escopo exigido)

Isso implementa o princípio de **Zero Trust**: nunca confiar, sempre verificar.

**Tratamento de AWS Lambdas**

⚠️ **Importante:** JWT funciona apenas se a lambda estiver exposta para ser consumida por HTTP Request. Temos muitas lambdas que são consumidas por Invoke direto.

**Lambda exposta via API Gateway (HTTP):**

- **Validação no API Gateway** (sem código na Lambda)
- API Gateway possui Authorizer nativo que valida JWT do Cognito
- Lambda recebe requisição já autenticada/autorizada

**Lambda consumida via Invoke (não HTTP):**

- Validação via IAM Policies
- Controle de acesso baseado em roles IAM (quem pode invocar)
- Não utiliza JWT (não é necessário para comunicação interna AWS)

**Prevenção de Compartilhamento de Credenciais**

**Problema:** Como garantir que o serviço X está utilizando suas próprias credenciais e não as de outro serviço?

**Temos algumas opções que tratam isso:**

**a) Template Helm atualizado - helm-aarin-base** ✅

- Injeção automática de credenciais via External Secrets Operator
- Path obrigatório: `/m2m/{environment}/{service-name}/client-secret`
- Cada pod recebe **apenas** as credenciais do seu próprio serviço — sem possibilidade de acesso às credenciais de outro serviço via Secrets Manager sem permissão IAM explícita

**b) Testes automatizados de arquitetura**

- Validar via testes de integração que o middleware de autorização está implementado (porém aqui teria que ser algo mais macro na pipe, pois o dev pode não fazer)

**c) Etapa no CI/CD** ✅ 

- Validar que a camada de autorização está implementada antes de permitir deploy
- Verificar presença de middleware JWT no código (script de validação a ser desenvolvido)
- Validar que credenciais corretas estão configuradas no Secrets Manager
- Verificar ausência de credenciais M2M hardcoded no código

**Gerenciamento de Segredos**

Credenciais dos App Clients ficam armazenadas no **AWS Secrets Manager** com expiração longa na Fase 1 — a rotação será feita manualmente conforme definição do time de Segurança (ex: anual ou semestral), seguindo o runbook documentado. A automação da rotação poderá ser avaliada em fases futuras.

**Atenção: Scopes não possuem hierarquia**

⚠️ **Ponto de atenção crítico:** Scopes do Cognito **não possuem hierarquia**.

**Exemplo do problema:**

Se você tem um endpoint com os seguintes scopes:

- `servico/all`
- `servico/criar-conta`

O usuário/serviço precisará ter **ambos os escopos** para acessar o endpoint.

**Solução para Fase 1:**

Utilizar **scopes genéricos** por contexto para simplificar:

- `core-context/all`
- `kyc-context/all`
- `ledger-context/all`
- `fee-context/all`

Granularidade adicional pode ser implementada futuramente (Fase 2) se necessário, ou via validação adicional no código da aplicação.

## **Observabilidade e Auditoria**

**Métricas no Datadog:**

- `m2m.token.emissao` por `client_id` — detecta serviços com cache mal configurado
- `m2m.jwt.falha_validacao` por serviço — detecta tokens inválidos ou expirados inesperadamente
- `m2m.jwt.violacao_escopo` — detecta serviços tentando acessar recursos sem permissão

**Dashboard Datadog** com painel de saúde M2M para o time de Plataforma/DevOps.

**Auditoria:** Todos os eventos de emissão de token são registrados automaticamente no **AWS CloudTrail** do Cognito (sem configuração adicional), com `client_id`, timestamp e escopos solicitados — habilitando rastreabilidade completa para fins de compliance.

---

## **Trade-offs e alternativas consideradas**

**Opção 1: Validação JWT no Código com AWS Cognito — Escolha para Fase 1**

**Arquitetura:**

- AWS Cognito para emissão de tokens M2M (substituição do Keycloak)
- Resource Servers por contexto de negócio
- **Bibliotecas internas** para obtenção de tokens (padronizamos implementação)
- **Validação JWT via código** em cada API
    - .NET: `Microsoft.AspNetCore.Authentication.JwtBearer`
    - Node.js: `aws-jwt-verify`
    - Lambda: Validação no API Gateway ou IAM Policies

**Prós:**

- ✅ Muito mais simples de operar
- ✅ Desenvolvedores já conhecem (middleware JWT padrão)
- ✅ Sem overhead de sidecars
- ✅ Menor complexidade operacional
- ✅ Implementação rápida (estimativa core-to-core: 3–6 meses)
- ✅ Ecossistema AWS nativo — sem dependência de SaaS externo
- ✅ Curva de aprendizado baixa (Cognito já é utilizado na Aarin)

**Contras:**

- ❌ Validação JWT vira responsabilidade de cada serviço (mitigado por bibliotecas + templates)
- ❌ Sem mTLS automático entre pods
- ❌ Sem canary/blue-green nativos
- ❌ Sem observabilidade de rede automática

**Por que escolhemos esta opção:**

- Resolve **80% dos problemas** (token único, rastreabilidade, revogação granular) com **20% da complexidade**
- Permite validar modelo de scopes antes de investir em service mesh
- Facilita migração futura para Istio (Fase 2) se necessário

---

## **Conclusão**

As definições nesta ADR propõem uma evolução significativa na arquitetura de autenticação e autorização entre microsserviços da Aarin, substituindo o modelo atual de **token único compartilhado** por uma solução robusta baseada em **OAuth 2.0 Client Credentials** com **AWS Cognito** como provedor de identidade centralizado.

**Decisão Arquitetural — Fase 1**

✅ **AWS Cognito M2M** como IdP centralizado (substituição do Keycloak)

✅ **Bibliotecas internas** (.NET, Node.js, Lambda) para abstração completa da autenticação

✅ **Validação JWT no código** de cada serviço (middleware padrão) com Zero Trust em dupla camada

✅ **Scopes globais** por contexto (`{contexto}/all`) para simplificar adoção inicial

✅ **Secrets Manager** para gerenciamento de credenciais M2M com expiração longa (rotação manual na Fase 1)

✅ **helm-aarin-base** + External Secrets Operator para prevenção de compartilhamento de credenciais

✅ **Observabilidade** via Datadog (métricas de emissão, falhas de validação e violações de escopo) e auditoria via AWS CloudTrail

**Justificativa:** Esta fase resolve **80% dos problemas** identificados (token único, falta de rastreabilidade, ausência de revogação granular) com complexidade operacional mínima e permite que a Aarin:

- Elimine o ponto único de falha de segurança atual
- Ganhe rastreabilidade completa de comunicações entre serviços
- Estabeleça processo de gestão de identidades M2M
- Padronize autenticação em bibliotecas reutilizáveis
- Valide o modelo de scopes antes de investir em Service Mesh

---

**Fase 2 — Evoluções Futuras**

Esta seção consolida todas as evoluções consideradas, mas que **não fazem parte do escopo da Fase 1**.

**Critério de reavaliação:** Reavaliar esta seção quando **todos** os gatilhos abaixo forem atendidos:

- ✅ 100% dos serviços do Core migrados para Cognito M2M (marco de conclusão da Fase 1)
- ✅ Bibliotecas internas estáveis (sem exceções no CI/CD)
- ✅ Ao menos um ciclo completo de rotação de segredos executado com sucesso

**Estimativa de janela para início da Fase 2:** 12 meses após o rollout completo da Fase 1.

**Service Mesh com Istio**

Adicionar Istio à malha de serviços para validação de identidade na camada de rede:

**Arquitetura complementar:**

- Tudo da Fase 1 +
- Istio Service Mesh para validação JWT na camada de rede (sidecar Envoy)
- mTLS automático entre pods
- Políticas de autorização centralizadas (`RequestAuthentication` + `AuthorizationPolicy`)
- Observabilidade de rede automática (traces, métricas de latência por serviço)

**Gatilhos que justificam adoção do Istio:**

- ✅ Crescimento significativo de microsserviços e necessidade de políticas uniformes
- ✅ Requisitos de compliance obrigatórios (SOC 2, PCI-DSS) exigindo mTLS obrigatório
- ✅ Necessidade comprovada de canary deployments, circuit breakers ou rate limiting na malha
- ✅ Equipe de Platform Engineering estabelecida (SREs dedicados)
- ✅ Maturidade em Kubernetes consolidada (troubleshooting avançado, GitOps maduro)

**Por que não agora:**

- Alta complexidade operacional e curva de aprendizado íngreme
- Requer equipe de SRE dedicada
- A Fase 1 já resolve os principais problemas de segurança e rastreabilidade

**Granularidade de Escopos**

Evoluir dos scopes genéricos por contexto (`{contexto}/all`) para scopes granulares por operação (`{contexto}/{operacao}`), permitindo controle de acesso mais fino. Requer validação de que o modelo de 100 escopos por Resource Server não será atingido e que o esforço de gestão é justificado pelo nível de controle necessário.

---

**Referências**

- [RFC 6749 — The OAuth 2.0 Authorization Framework](https://datatracker.ietf.org/doc/html/rfc6749)
- [AWS Cognito — Client Credentials Grant](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-userpools-server-contract-reference.html)
- [AWS Cognito — Resource Servers and Scopes](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-define-resource-servers.html)
- [AWS Cognito — Service Quotas e Limites](https://docs.aws.amazon.com/cognito/latest/developerguide/limits.html)
- [AWS Secrets Manager — Gerenciamento de Segredos](https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html)
- [aws-jwt-verify — Biblioteca Oficial AWS para Validação JWT](https://github.com/awslabs/aws-jwt-verify)
- [Istio — Security](https://istio.io/latest/docs/concepts/security/)
- [Istio — Authorization Policy](https://istio.io/latest/docs/reference/config/security/authorization-policy/)
- [Istio — Request Authentication](https://istio.io/latest/docs/reference/config/security/request_authentication/)
- [NIST — Zero Trust Architecture (SP 800-207)](https://csrc.nist.gov/publications/detail/sp/800-207/final)
- [IdentityModel — .NET OAuth 2.0 Client Library](https://github.com/IdentityModel/IdentityModel)
