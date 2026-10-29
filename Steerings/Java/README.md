# Steerings — Java

Convenções de desenvolvimento para projetos **Java** com arquitetura em camadas (Clean / Ports & Adapters) usando Spring Boot.

Forneça estes documentos como contexto a agentes de IA antes de gerar ou revisar código Java, ou leia-os diretamente como guia de estilo da equipe.

---

## Estrutura

```text
Java/
├── general/       # Convenções gerais (arquitetura, código, cache) + teste geral
├── application/   # Camada Application: convenção + teste
├── domain/        # Camada Domain: convenção + teste
├── external/      # Camada External: convenção + testes (persistence, gateway, security)
└── presentation/  # Camada Presentation: convenção + testes (controller, scheduler)
```

---

## Documentos disponíveis

### general/

| Documento | Conteúdo |
|-----------|----------|
| [`general/convention-architecture.md`](./general/convention-architecture.md) | Princípios arquiteturais, estrutura de pastas (`core/`, `configuration/`), fluxo de dependências e de dados, responsabilidades de cada camada. |
| [`general/convention-code.md`](./general/convention-code.md) | Nomenclatura, constantes, formatação em blocos lógicos, ordenação de membros, injeção de dependência, logging, configurações tipadas, imports e análise estática. |
| [`general/convention-test-general.md`](./general/convention-test-general.md) | Layout, fixtures, padrão AAA, `@DisplayName`, AssertJ, Testcontainers, WireMock. |

### application/

| Documento | Conteúdo |
|-----------|----------|
| [`application/convention-layer-application.md`](./application/convention-layer-application.md) | **Application** — casos de uso, orquestração e definição de contratos (portas). |
| [`application/convention-test-layer-application.md`](./application/convention-test-layer-application.md) | Testes da camada Application. |

### domain/

| Documento | Conteúdo |
|-----------|----------|
| [`domain/convention-layer-domain.md`](./domain/convention-layer-domain.md) | **Domain** — entidades, value objects, `Result<T>`, validações e exceções de negócio. Independente de frameworks. |
| [`domain/convention-test-layer-domain.md`](./domain/convention-test-layer-domain.md) | Testes da camada Domain. |

### external/

| Documento | Conteúdo |
|-----------|----------|
| [`external/convention-layer-external.md`](./external/convention-layer-external.md) | **External** — princípios transversais dos adaptadores de infraestrutura. |
| [`external/convention-layer-external-persistence.md`](./external/convention-layer-external-persistence.md) | Detalhes de persistência (DAO, Repository, Mapper, Model). |
| [`external/convention-layer-external-gateway.md`](./external/convention-layer-external-gateway.md) | Integração HTTP com outras aplicações (Feign Client + Resilience4j). |
| [`external/convention-layer-external-cache.md`](./external/convention-layer-external-cache.md) | Cache (Caffeine / Redis): onde aplicar, invalidação, serialização e configuração. |
| [`external/convention-layer-external-security.md`](./external/convention-layer-external-security.md) | Segurança dentro da camada External. |
| [`external/convention-test-layer-external.md`](./external/convention-test-layer-external.md) | Testes da camada External. |
| [`external/convention-test-layer-external-persistence.md`](./external/convention-test-layer-external-persistence.md) | Testes de persistência. |
| [`external/convention-test-layer-external-gateway.md`](./external/convention-test-layer-external-gateway.md) | Testes de Gateway HTTP (adaptador Feign). |

### presentation/

| Documento | Conteúdo |
|-----------|----------|
| [`presentation/convention-layer-presentation.md`](./presentation/convention-layer-presentation.md) | **Presentation** — princípios transversais de acesso à aplicação. |
| [`presentation/convention-layer-presentation-controller.md`](./presentation/convention-layer-presentation-controller.md) | Acesso HTTP: controllers, payloads, Global Exception Handler e OpenAPI. |
| [`presentation/convention-layer-presentation-scheduler.md`](./presentation/convention-layer-presentation-scheduler.md) | Acesso via Scheduler com `@Scheduled` e ShedLock. |
| [`presentation/convention-test-layer-presentation-controller.md`](./presentation/convention-test-layer-presentation-controller.md) | Testes de Controller (`@WebMvcTest` / integração). |
| [`presentation/convention-test-layer-presentation-scheduler.md`](./presentation/convention-test-layer-presentation-scheduler.md) | Testes de Scheduler, unitários e sem contexto Spring. |

---

## Fluxo de dependências

```text
Presentation  →  Application  →  Domain
External      →  Application  →  Domain
```

- **Domain**: não depende de nenhuma outra camada.
- **Application**: depende apenas de Domain.
- **External**: depende de Application e Domain.
- **Presentation**: depende de Application.

---

## Guia rápido por tarefa

| Tarefa | Documentos recomendados |
|--------|------------------------|
| Implementar entidade | `general/convention-architecture.md` + `domain/convention-layer-domain.md` |
| Testar entidade / domínio | `general/convention-test-general.md` + `domain/convention-test-layer-domain.md` |
| Implementar caso de uso | `application/convention-layer-application.md` |
| Testar caso de uso | `general/convention-test-general.md` + `application/convention-test-layer-application.md` |
| Implementar controller | `presentation/convention-layer-presentation.md` + `presentation/convention-layer-presentation-controller.md` |
| Testar controller | `general/convention-test-general.md` + `presentation/convention-test-layer-presentation-controller.md` |
| Implementar scheduler | `presentation/convention-layer-presentation.md` + `presentation/convention-layer-presentation-scheduler.md` |
| Testar scheduler | `general/convention-test-general.md` + `presentation/convention-test-layer-presentation-scheduler.md` |
| Implementar persistência | `external/convention-layer-external.md` + `external/convention-layer-external-persistence.md` |
| Testar persistência | `general/convention-test-general.md` + `external/convention-test-layer-external-persistence.md` |
| Implementar gateway HTTP | `external/convention-layer-external.md` + `external/convention-layer-external-gateway.md` |
| Testar gateway HTTP | `general/convention-test-general.md` + `external/convention-test-layer-external-gateway.md` |
| Implementar cache | `external/convention-layer-external-cache.md` |
| Revisão geral | Conjunto completo |
