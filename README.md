# Documentação de Arquitetura e Melhores Práticas

> Guia de padrões e boas práticas para as APIs .NET Core da plataforma.

---

## Índice

- [Visão Geral da Arquitetura](#visão-geral-da-arquitetura)
- [Fluxo da Aplicação](#fluxo-da-aplicação)
- [Stack Tecnológica](#stack-tecnológica)
- [Documentação Detalhada](#documentação-detalhada)

---

## Visão Geral da Arquitetura

A plataforma adota uma arquitetura em camadas com separação clara de responsabilidades, garantindo escalabilidade, resiliência e facilidade de manutenção.

```
Clientes Externos
(Mobile / Web / WhatsApp)
         │
         ▼
┌─────────────────────────┐
│  SENSEDIA (Proxy Externo)│   ◄── Gerenciamento de APIs, rate limiting,
│    Gateway / Proxy       │        autenticação externa, throttling
└─────────────┬───────────┘
              │ REST via HTTPS
              ▼
┌─────────────────────────────────────────────────────────────────┐
│                        K8S CLUSTER                              │
│                                                                 │
│   ┌──────────────────────┐      ┌──────────────────────────┐   │
│   │    BFF API .NET       │◄────►│     CORE API .NET        │   │
│   │  (Backend for        │      │  (Domínio / Negócio)     │   │
│   │   Frontend)          │      │                          │   │
│   └──────────────────────┘      └──────────────┬───────────┘   │
│                                                │               │
└────────────────────────────────────────────────┼───────────────┘
                                                 │ gRPC / REST
                                                 ▼
                              ┌──────────────────────────────────┐
                              │              GCP                  │
                              │  Cloud Storage │ Firestore        │
                              │  Pub/Sub       │ Data Store       │
                              └──────────────────────────────────┘

Serviços Internos (via Sensedia Interno):
  ├── Orquestrador de Pagamentos
  ├── Key Cloak (IAM/Auth)
  └── REST ASA Legado

Público exposto na Internet (K8S):
  ├── Incognia
  └── ASA API
```

---

## Fluxo da Aplicação

```
externo → proxy → BFF .NET Core → Core API .NET Core → GCP
```

| Etapa | Componente | Responsabilidade |
|-------|-----------|-----------------|
| 1 | Cliente (Mobile/Web/WhatsApp) | Origina a requisição |
| 2 | Sensedia Externo (Proxy) | Gateway de entrada, autenticação, rate limit |
| 3 | BFF API .NET Core | Orquestra e adapta respostas para cada canal |
| 4 | Core API .NET Core | Regras de negócio, domínio e persistência |
| 5 | GCP | Armazenamento, mensageria e dados |

---

## Stack Tecnológica

| Tecnologia | Uso |
|------------|-----|
| **.NET 10** | Runtime das APIs BFF e Core |
| **Clean Architecture** | Estrutura de projeto |
| **REST via HTTPS** | Comunicação entre serviços |
| **gRPC** | Comunicação interna entre BFF e Core |
| **Dapper / Entity Framework** | ORM para acesso a dados |
| **xUnit** | Testes unitários e integrados |
| **K8S** | Orquestração de containers |
| **GCP** | Cloud: Storage, Firestore, Pub/Sub |
| **Sensedia** | API Gateway (proxy externo e interno) |
| **KeyCloak** | Identity & Access Management |

---

## Documentação Detalhada

| Documento | Descrição |
|-----------|-----------|
| [Arquitetura](docs/architecture.md) | Diagrama e detalhamento dos componentes |
| [Clean Architecture](docs/clean-architecture.md) | Padrão de estrutura dos projetos .NET |
| [Padrões de API REST](docs/api-standards.md) | Convenções de endpoints, versionamento e contratos |
| [ORM – Dapper & EF Core](docs/orm.md) | Guia de uso dos ORMs e acesso a dados |
| [Testes](docs/testing.md) | Estratégia, padrões e exemplos de testes |
| [Boas Práticas](docs/best-practices.md) | SOLID, Clean Code e princípios gerais |
