# Visão Geral da Arquitetura

## Fluxo Principal

```
Cliente / Externo
       │
       ▼
   [ Proxy / API Gateway ]
       │  (roteamento, TLS termination, rate limiting)
       ▼
   [ BFF — Backend for Frontend (.NET Core 10) ]
       │  (agregação, adaptação de contratos, autenticação)
       ▼
   [ API Core (.NET Core 10) ]
       │  (domínio, regras de negócio, persistência)
       ▼
   [ GCP — Google Cloud Platform ]
       (Cloud SQL, Pub/Sub, Cloud Storage, Secret Manager, etc.)
```

---

## Responsabilidade de Cada Camada

### 1. Proxy / API Gateway

| Responsabilidade | Exemplos de ferramentas |
|---|---|
| TLS termination | Cloud Load Balancing, NGINX |
| Rate limiting e throttling | NGINX, Apigee |
| Autenticação de borda (JWT / mTLS) | Apigee, Cloud Endpoints |
| Roteamento para os BFFs | Cloud Load Balancing |
| Observabilidade (logs, tracing) | Cloud Trace, Datadog |

**Boas práticas:**
- Nunca implemente lógica de negócio no proxy.
- Utilize health-checks (`/health`) e readiness probes em todos os serviços.
- Centralize políticas de CORS no proxy, não nos serviços internos.
- Configure timeouts agressivos no edge (ex.: 30 s) para proteger os recursos internos.

---

### 2. BFF — Backend for Frontend

O BFF é a fachada orientada ao cliente (web, mobile, parceiros). Cada canal pode ter seu próprio BFF.

**Responsabilidades:**
- Agregar chamadas a múltiplas APIs Core em uma única resposta.
- Adaptar o contrato de resposta ao formato esperado pelo cliente.
- Validar e propagar tokens de autenticação/autorização.
- Não conter regras de negócio — apenas orquestração e adaptação.

**Boas práticas:**
- Versione os endpoints (`/v1/`, `/v2/`).
- Use `HttpClientFactory` com `Polly` para resiliência (retry, circuit breaker).
- Serialize/deserialize com `System.Text.Json` configurado de forma centralizada.
- Implemente circuit breaker para chamadas às APIs Core.

```csharp
// Exemplo: registro do HttpClient com Polly no BFF
builder.Services.AddHttpClient<IOrderApiClient, OrderApiClient>(client =>
{
    client.BaseAddress = new Uri(builder.Configuration["ApiCore:Orders:BaseUrl"]!);
    client.Timeout = TimeSpan.FromSeconds(10);
})
.AddResilienceHandler("default", pipeline =>
{
    pipeline.AddRetry(new HttpRetryStrategyOptions
    {
        MaxRetryAttempts = 3,
        Delay = TimeSpan.FromMilliseconds(200),
        BackoffType = DelayBackoffType.Exponential
    });
    pipeline.AddCircuitBreaker(new HttpCircuitBreakerStrategyOptions
    {
        FailureRatio = 0.5,
        SamplingDuration = TimeSpan.FromSeconds(10),
        BreakDuration = TimeSpan.FromSeconds(30)
    });
});
```

---

### 3. API Core

Serviços internos que encapsulam o domínio e as regras de negócio. Seguem o padrão **Clean Architecture** (ver [clean-architecture.md](clean-architecture.md)).

**Responsabilidades:**
- Expor endpoints REST para o BFF.
- Executar regras de negócio no domínio.
- Persistir e recuperar dados via ORM (Dapper / Entity Framework).
- Publicar/consumir eventos no GCP Pub/Sub quando necessário.

**Boas práticas:**
- Não exponha diretamente ao cliente externo — apenas o BFF os consome.
- Utilize autenticação mTLS ou tokens de serviço internos (service-to-service).
- Aplique Clean Architecture com separação em camadas (Domain, Application, Infrastructure, API).

---

### 4. GCP — Google Cloud Platform

| Serviço GCP | Uso |
|---|---|
| Cloud SQL (PostgreSQL / SQL Server) | Persistência relacional |
| Cloud Pub/Sub | Mensageria assíncrona entre serviços |
| Cloud Storage | Armazenamento de arquivos e blobs |
| Secret Manager | Gerenciamento de segredos e connection strings |
| Cloud Run / GKE | Hospedagem dos containers .NET |
| Cloud Trace / Cloud Logging | Observabilidade distribuída |

**Boas práticas:**
- Nunca armazene segredos em variáveis de ambiente ou `appsettings.json`. Use o **Secret Manager**.
- Configure o `Workload Identity` para que os pods no GKE autentiquem no GCP sem chaves de serviço.
- Utilize VPC Service Controls para isolar os serviços internos.
- Defina alertas e SLOs via Cloud Monitoring.

---

## Diagrama de Implantação (resumido)

```
Internet
   │
[ Cloud Load Balancing + SSL ]
   │
[ API Gateway / NGINX Ingress ]
   │
┌──┴──────────────────────────────────────────┐
│  GKE Cluster (VPC privada)                   │
│                                              │
│  [ BFF Web ]  [ BFF Mobile ]                 │
│       │             │                        │
│       └──────┬──────┘                        │
│              │                               │
│       [ API Core — Orders ]                  │
│       [ API Core — Catalog ]                 │
│       [ API Core — Users ]                   │
│              │                               │
│       [ Cloud SQL ] [ Pub/Sub ]              │
└──────────────────────────────────────────────┘
```

---

## Princípios gerais

1. **Separation of Concerns** — cada camada tem uma única razão para existir.
2. **Defense in Depth** — autenticação em múltiplas camadas (edge, BFF, API Core).
3. **Fail Fast** — valide entradas o mais cedo possível na pipeline.
4. **Observabilidade** — logs estruturados (JSON), distributed tracing e métricas em todas as camadas.
5. **Infrastructure as Code** — Terraform ou Config Connector para provisionar recursos GCP.
