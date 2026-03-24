# Arquitetura da Plataforma

## Diagrama de Componentes

![Arquitetura](https://github.com/user-attachments/assets/ab739237-c117-4d4c-8dd3-deca99a4fa40)

---

## Componentes

### Clientes Externos

Os clientes que iniciam as requisições para a plataforma:

| Cliente | Protocolo |
|---------|-----------|
| Mobile (iOS / Android) | REST via HTTPS |
| Web (Browser / SPA) | REST via HTTPS |
| WhatsApp (Chatbot) | REST via HTTPS |

Todos os clientes se comunicam exclusivamente através do **Sensedia Externo (Proxy)**, nunca diretamente com os serviços internos.

---

### Sensedia Externo – Proxy (API Gateway)

Responsável por:

- Autenticação e autorização de clientes externos (OAuth 2.0 / JWT via KeyCloak)
- Rate limiting e throttling
- Roteamento de requisições para o BFF correspondente
- Log e rastreamento (tracing) de requisições de entrada
- Transformação de protocolos quando necessário
- Proteção contra ataques (WAF, DDoS)

**Tecnologia:** Sensedia API Platform

---

### K8S Cluster

Ambiente de execução dos serviços internos. Garante:

- Alta disponibilidade com múltiplas réplicas
- Auto-scaling horizontal (HPA)
- Rolling deployments sem downtime
- Isolamento de rede entre serviços (Network Policies)
- Service mesh para observabilidade e controle de tráfego

#### BFF API .NET Core (Backend for Frontend)

O BFF é um padrão onde cada canal de consumo (Mobile, Web) possui seu próprio backend especializado que:

- Agrega chamadas a múltiplos serviços internos em uma única resposta
- Adapta o contrato de resposta ao formato ideal para cada cliente
- Gerencia autenticação e sessão do usuário final
- Implementa cache de curto prazo para respostas frequentes
- Não contém regras de negócio — apenas orquestração e adaptação

**Comunicação:**
- Recebe: REST via HTTPS (do Proxy)
- Envia para Core: REST via HTTPS ou gRPC (comunicação interna)

**Estrutura esperada:** [Clean Architecture](clean-architecture.md)

#### Core API .NET Core

Contém toda a lógica de negócio da plataforma:

- Domínio rico com Entidades, Value Objects e Agregados
- Casos de uso (Application Layer / Use Cases)
- Portas e adaptadores para serviços externos (GCP, Legacy)
- Persistência via Dapper ou Entity Framework Core
- Publicação de eventos no Pub/Sub do GCP

**Comunicação:**
- Recebe: REST via HTTPS ou gRPC (do BFF)
- Envia para GCP: gRPC / SDK do GCP
- Envia para Serviços Internos: REST via Sensedia Interno

---

### GCP – Google Cloud Platform

Infraestrutura de dados e mensageria:

| Serviço GCP | Uso |
|-------------|-----|
| **Cloud Storage** | Armazenamento de arquivos e blobs |
| **Cloud Storage for Firebase** | Armazenamento para apps mobile/web via Firebase |
| **Firestore / Data Store** | Banco NoSQL para dados não relacionais |
| **Pub/Sub** | Mensageria assíncrona entre serviços |

A Core API é o único componente que interage diretamente com o GCP.

---

### Sensedia Interno – Serviços de Terceiros

Gateway para serviços externos parceiros:

| Serviço | Responsabilidade |
|---------|-----------------|
| **Orquestrador de Pagamentos** | Processamento e orquestração de transações financeiras |
| **KeyCloak** | Identity & Access Management, emissão de tokens JWT |
| **REST ASA Legado** | Integração com sistema legado via REST |

A comunicação com esses serviços ocorre sempre via **Sensedia Interno**, garantindo governança, segurança e rastreabilidade.

---

### Exposição Pública na Internet (K8S)

Serviços expostos diretamente na internet via K8S Ingress:

| Serviço | Descrição |
|---------|-----------|
| **Incognia** | Detecção de fraude e verificação de dispositivo |
| **ASA API** | API pública para integrações externas autorizadas |

---

## Fluxo de Comunicação Detalhado

```
1. Cliente → Sensedia Externo
   Protocolo: HTTPS (TLS 1.3)
   Auth: Bearer Token (JWT emitido pelo KeyCloak)

2. Sensedia Externo → BFF API .NET
   Protocolo: REST via HTTPS
   Headers: X-Correlation-ID, X-User-ID, Authorization

3. BFF API .NET → Core API .NET
   Protocolo: REST via HTTPS (sincrono) ou gRPC (alta performance)
   Comunicação interna via K8S Service DNS

4. Core API .NET → GCP
   Protocolo: gRPC (SDK Google) / HTTPS
   Auth: Service Account (Workload Identity)

5. Core API .NET → Sensedia Interno
   Protocolo: REST via HTTPS
   Auth: mTLS + OAuth2 Client Credentials
```

---

## Princípios de Observabilidade

Todos os serviços devem implementar:

- **Logging estruturado** em JSON (compatível com Cloud Logging do GCP)
- **Tracing distribuído** com OpenTelemetry e Correlation ID propagado em todos os headers
- **Métricas** expostas via `/metrics` (Prometheus-compatible)
- **Health checks** em `/health/live` e `/health/ready`

```csharp
// Exemplo de configuração de Observabilidade no Program.cs
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing => tracing
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddGrpcClientInstrumentation()
        .AddOtlpExporter())
    .WithMetrics(metrics => metrics
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddPrometheusExporter());
```
