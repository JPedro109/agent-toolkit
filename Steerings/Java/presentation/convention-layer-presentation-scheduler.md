# Convenções da Camada Presentation — Scheduler

Este documento detalha o adaptador de entrada **Scheduler** (tarefas agendadas) da camada Presentation. Ele é uma extensão do documento base `convention-layer-presentation.md` e assume **todos** os princípios transversais definidos lá (sem regra de negócio, dependência exclusiva da camada Application, orquestração em quatro passos, tratamento de erro na borda e observabilidade).

Um scheduler é apenas mais uma forma de acesso à aplicação: em vez de uma requisição HTTP, o estímulo de entrada é o tempo (um gatilho de agendamento). Ele converte o estímulo em uma invocação de caso de uso, executa e trata o resultado — exatamente como um controller, sem HTTP.

---

# Estrutura

Tarefas agendadas residem em seu próprio subpacote dentro de `presentation/`, irmão de `controller/`.

```text
presentation/
├── controller/
└── scheduler/
    └── SampleScheduler.java
```

> A habilitação do agendamento e do lock distribuído (`@EnableScheduling`, `@EnableSchedulerLock`) e o bean `LockProvider` são **configuração técnica de bootstrap**: residem no pacote transversal `configuration/`, fora de `core/`, conforme o documento de arquitetura. O subpacote `scheduler/` contém apenas as classes de tarefa.

As seguintes regras devem ser respeitadas:

- Cada responsabilidade agendada deve possuir sua própria classe de scheduler.
- O scheduler não contém regra de negócio — delega a um caso de uso, igual ao controller.
- Componentes compartilhados entre schedulers devem permanecer em `common`.

---

# Agendamento

O agendamento utiliza o suporte nativo do Spring.

As seguintes regras devem ser respeitadas:

- Habilitar o agendamento com `@EnableScheduling` (na configuração de bootstrap, em `configuration/`).
- Cada tarefa é um método anotado com `@Scheduled`.
- Preferir `cron`/`fixedDelayString`/`fixedRateString` apontando para uma propriedade externa (`@Scheduled(cron = "${sample.scheduler.cron}")`) em vez de valores fixos no código, para permitir ajuste por ambiente.
- O método agendado deve ser pequeno e objetivo: logar o início, montar o Input DTO, executar o caso de uso e logar o fim — sem HTTP e sem regra de negócio.

---

# Lock Distribuído

Em ambiente com múltiplas instâncias, uma tarefa `@Scheduled` dispararia em cada instância simultaneamente. Para garantir execução **no máximo uma vez** por gatilho, o lock distribuído é **obrigatório**.

As seguintes regras devem ser respeitadas:

- Utilizar **ShedLock** para o lock distribuído.
- Habilitar o lock com `@EnableSchedulerLock` na configuração de bootstrap, definindo um `defaultLockAtMostFor` de segurança.
- Anotar cada tarefa com `@SchedulerLock`, informando `name` único, `lockAtMostFor` (tempo máximo que o lock é mantido caso a instância morra sem liberar) e `lockAtLeastFor` (tempo mínimo, evita reexecução quando a tarefa é muito rápida).
- O **provedor de lock** (`LockProvider`) é uma **escolha de infraestrutura do projeto** (JDBC, Redis, Mongo, etc.) e é declarado como bean em `configuration/`. A convenção não impõe um provedor — apenas exige que exista um. Veja exemplos na seção "Provedor de Lock".

## ✔ Correto

```java
@Component
public class SampleScheduler {

    private static final Logger LOGGER = LoggerFactory.getLogger(SampleScheduler.class);

    private final ProcessSampleUseCase processSampleUseCase;

    public SampleScheduler(ProcessSampleUseCase processSampleUseCase) {
        this.processSampleUseCase = processSampleUseCase;
    }

    @Scheduled(cron = "${sample.scheduler.cron}")
    @SchedulerLock(
            name = "SampleScheduler_processPending",
            lockAtMostFor = "PT10M",
            lockAtLeastFor = "PT1M"
    )
    public void processPending() {
        LOGGER.info("Processing pending samples - started");

        processSampleUseCase.execute();

        LOGGER.info("Processing pending samples - finished");
    }
}
```

## ❌ Incorreto — regra de negócio no scheduler

```java
@Component
public class SampleScheduler {

    @Scheduled(fixedRate = 60000)
    public void processPending() {
        var samples = repository.findPending();
        for (var sample : samples) {
            if (sample.isExpired()) {
                sample.cancel();
            }
        }
    }
}
```

## ❌ Incorreto — tarefa agendada sem lock distribuído

```java
@Component
public class SampleScheduler {

    @Scheduled(cron = "${sample.scheduler.cron}")
    public void processPending() {
        processSampleUseCase.execute();
    }
}
```

---

# Configuração do Lock

A habilitação do ShedLock vive na configuração de bootstrap transversal.

```java
@Configuration
@EnableScheduling
@EnableSchedulerLock(defaultLockAtMostFor = "PT30S")
public class SchedulerConfig { }
```

---

# Provedor de Lock

O `LockProvider` define **onde** o lock é armazenado. A escolha é do projeto, conforme a infraestrutura já disponível. Declare **um** bean `LockProvider` em `configuration/`. Abaixo, exemplos dos provedores mais comuns — escolha o que fizer sentido para o seu ambiente.

## JDBC (banco relacional)

```java
@Bean
public LockProvider lockProvider(DataSource dataSource) {
    return new JdbcTemplateLockProvider(
            JdbcTemplateLockProvider.Configuration.builder()
                    .withJdbcTemplate(new JdbcTemplate(dataSource))
                    .usingDbTime()
                    .build()
    );
}
```

> Requer a tabela `shedlock`. Crie-a via migração (Flyway), conforme a convenção de persistência.

## Redis

```java
@Bean
public LockProvider lockProvider(RedisConnectionFactory connectionFactory) {
    return new RedisLockProvider(connectionFactory);
}
```

## MongoDB

```java
@Bean
public LockProvider lockProvider(MongoClient mongoClient) {
    return new MongoLockProvider(mongoClient.getDatabase("sample"));
}
```

---

# Dependências

Valem as mesmas regras de dependência da camada Presentation (ver documento base): o scheduler depende apenas da camada Application e nunca acessa repositórios, adaptadores, DAOs, modelos de persistência ou componentes da External.

---

# Resumo das Convenções

Além dos princípios transversais de `convention-layer-presentation.md`, o acesso via Scheduler deve respeitar:

- Cada responsabilidade agendada possui sua própria classe de scheduler, em `presentation/scheduler/`.
- O agendamento usa `@EnableScheduling` + `@Scheduled`, preferindo expressões externalizadas por propriedade.
- O lock distribuído é obrigatório via **ShedLock**: `@EnableSchedulerLock` na configuração e `@SchedulerLock` em cada tarefa.
- Toda tarefa define `name`, `lockAtMostFor` e `lockAtLeastFor`.
- O `LockProvider` é escolha de infraestrutura do projeto (JDBC, Redis, Mongo, etc.), declarado como bean em `configuration/`.
- O scheduler não implementa regra de negócio — apenas orquestra um caso de uso.
