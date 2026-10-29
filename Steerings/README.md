# Steerings

Esta pasta contém os **documentos de convenção** que servem como contexto para agentes de IA. Cada arquivo Markdown descreve uma convenção específica de arquitetura, código ou teste para uma linguagem.

Os documentos são agnósticos de ferramenta e podem ser fornecidos como contexto a qualquer agente de IA que aceite Markdown, ou lidos diretamente pela equipe como guia de estilo.

---

## Estrutura

```text
Steerings/
├── Java/
│   ├── general/       # Convenções gerais (arquitetura, código, cache) + teste geral
│   ├── application/   # Camada Application: convenção + teste
│   ├── domain/        # Camada Domain: convenção + teste
│   ├── external/      # Camada External: convenção + testes (persistence, gateway, security)
│   └── presentation/  # Camada Presentation: convenção + testes (controller, scheduler)
└── Kotlin/
    ├── general/
    ├── application/
    ├── domain/
    ├── external/
    └── presentation/
```

Cada pasta agrupa a convenção da camada junto com seus testes correspondentes. O conjunto de documentos é **espelhado** entre Java e Kotlin — ao alterar uma convenção, atualize as duas versões para mantê-las alinhadas.

> **Extensibilidade:** novas linguagens podem ser adicionadas criando uma subpasta com a mesma estrutura de pastas. Basta copiar de `Java/` ou `Kotlin/`, adaptar os exemplos de código e registrar na tabela de [Subpastas](#subpastas) abaixo.

---

## Convenções disponíveis

### general/

| Documento | Conteúdo |
|-----------|----------|
| `convention-architecture.md` | Princípios arquiteturais, estrutura de pastas, fluxo de dependências e responsabilidades de cada camada. |
| `convention-code.md` | Nomenclatura, formatação, ordenação de membros, injeção de dependência, logging, imports e análise estática. |
| `convention-cache.md` | Convenções de cache. |
| `convention-test-general.md` | Layout dos testes, fixtures, padrão AAA, `@DisplayName`, AssertJ, Testcontainers e WireMock. |

### application/

| Documento | Conteúdo |
|-----------|----------|
| `convention-layer-application.md` | **Application** — casos de uso, orquestração e definição de contratos (portas). |
| `convention-test-layer-application.md` | Testes da camada Application. |

### domain/

| Documento | Conteúdo |
|-----------|----------|
| `convention-layer-domain.md` | **Domain** — entidades, value objects, `Result<T>`, validações e exceções de negócio. Independente de frameworks. |
| `convention-test-layer-domain.md` | Testes da camada Domain. |

### external/

| Documento | Conteúdo |
|-----------|----------|
| `convention-layer-external.md` | **External** — princípios transversais dos adaptadores de infraestrutura. |
| `convention-layer-external-persistence.md` | Detalhes de persistência (DAO, Repository, Mapper, Model). |
| `convention-layer-external-gateway.md` | Integração HTTP com outras aplicações (Feign Client + Resilience4j). |
| `convention-layer-external-security.md` | Segurança dentro da camada External. |
| `convention-test-layer-external.md` | Testes da camada External. |
| `convention-test-layer-external-persistence.md` | Testes de persistência. |
| `convention-test-layer-external-gateway.md` | Testes de Gateway HTTP (adaptador Feign). |

### presentation/

| Documento | Conteúdo |
|-----------|----------|
| `convention-layer-presentation.md` | **Presentation** — princípios transversais de acesso à aplicação. |
| `convention-layer-presentation-controller.md` | Acesso HTTP: controllers, payloads, Global Exception Handler e OpenAPI. |
| `convention-layer-presentation-scheduler.md` | Acesso via Scheduler com `@Scheduled` e ShedLock. |
| `convention-test-layer-presentation-controller.md` | Testes de Controller (`@WebMvcTest` / integração). |
| `convention-test-layer-presentation-scheduler.md` | Testes de Scheduler, unitários e sem contexto Spring. |

---

## Como fornecer como contexto a um agente

Aponte os documentos relevantes para a tarefa em andamento:

- Implementando uma entidade → `general/convention-architecture.md` + `domain/convention-layer-domain.md`
- Criando um caso de uso → `application/convention-layer-application.md`
- Escrevendo um controller → `presentation/convention-layer-presentation.md` + `presentation/convention-layer-presentation-controller.md`
- Escrevendo testes → `general/convention-test-general.md` + documento de teste da camada correspondente
- Revisão geral → forneça o conjunto completo da linguagem

---

## Subpastas

| Pasta | README |
|-------|--------|
| [`Java/`](./Java/README.md) | Convenções para projetos Java |
| [`Kotlin/`](./Kotlin/README.md) | Convenções para projetos Kotlin |
