# Testes da Camada Presentation — Scheduler

Os testes de **Scheduler** têm como objetivo garantir que a tarefa agendada orquestre corretamente o caso de uso correspondente.

Eles seguem os mesmos princípios dos testes de Controller (ver `convention-test-layer-presentation-controller.md`), com uma diferença central: como o scheduler **não tem contrato HTTP**, o teste é **unitário puro** — sem contexto Spring, sem slice web. O gatilho de tempo (`@Scheduled`) e o lock distribuído (`@SchedulerLock`) são responsabilidade do framework e da configuração de bootstrap; o teste não os exercita. O que se testa é o **corpo do método agendado**: que ele delega ao caso de uso, sem regra de negócio.

---

## Estrutura

Os testes devem permanecer organizados conforme o scheduler.

Exemplo:

```text
src/test/java/
└── core/
    └── presentation/
        └── scheduler/
            └── SampleSchedulerTest.java
```

---

## Convenções

As seguintes regras devem ser respeitadas:

- Cada Scheduler deve possuir sua própria classe de teste.
- O teste é unitário: a classe de scheduler é instanciada diretamente, com o caso de uso mockado via Mockito.
- Nunca subir contexto Spring (`@SpringBootTest`, `@WebMvcTest`, etc.): não há nada de Spring para exercitar no corpo da tarefa.
- Não testar o agendamento em si (`@Scheduled`) nem o lock (`@SchedulerLock`) — são responsabilidade do framework/configuração.
- Todo método público (tarefa) do scheduler deve possuir teste.
- Deve ser validado que o caso de uso é invocado, com os argumentos corretos quando houver.
- Utilizar Fixtures para construção de quaisquer DTOs de entrada/saída dos casos de uso.
- Os testes devem utilizar AssertJ e os verificadores do Mockito.
- Todo método deve possuir `@DisplayName`.
- Os testes devem seguir o padrão AAA (Arrange → Act → Assert).

---

## Dependências

Os testes de Scheduler devem utilizar apenas:

- Mockito;
- Fixtures;
- AssertJ.

Não devem utilizar:

- Contexto Spring;
- Banco de dados;
- Repositories;
- Adaptadores da External;
- Casos de uso reais.

---

## Cenários obrigatórios

Todo Scheduler deve possuir, no mínimo, testes para:

- Execução da tarefa: o caso de uso correspondente é invocado uma vez.
- Propagação de exceção: se o caso de uso falha, a tarefa não engole a exceção (deixa o erro propagar para o tratamento padrão do agendador).

---

## ✔ Correto

```java
@ExtendWith(MockitoExtension.class)
class SampleSchedulerTest {

    @Mock
    private ProcessSampleUseCase processSampleUseCase;

    @InjectMocks
    private SampleScheduler sampleScheduler;

    @Test
    @DisplayName("Should invoke use case when processing pending samples")
    void shouldInvokeUseCaseWhenProcessingPending() {
        sampleScheduler.processPending();

        verify(processSampleUseCase).execute();
    }

    @Test
    @DisplayName("Should propagate exception when use case fails")
    void shouldPropagateExceptionWhenUseCaseFails() {
        doThrow(new IllegalStateException("boom"))
                .when(processSampleUseCase).execute();

        assertThatThrownBy(() -> sampleScheduler.processPending())
                .isInstanceOf(IllegalStateException.class);
    }
}
```

---

## ❌ Incorreto — subir contexto Spring para testar a tarefa

```java
@SpringBootTest
class SampleSchedulerTest {

    @Autowired
    private SampleScheduler sampleScheduler;
}
```

---

## ❌ Incorreto — testar o agendador em vez da orquestração

```java
@Test
void shouldRunEveryMinute() {
    // Verificar o intervalo do @Scheduled é testar o framework, não a tarefa.
}
```

---

## Resumo das Convenções

Todo teste de Scheduler deve respeitar os seguintes princípios:

- Cada scheduler possui sua própria classe de teste.
- O teste é unitário puro, com o caso de uso mockado — sem contexto Spring.
- Não se testa `@Scheduled` nem `@SchedulerLock` (responsabilidade do framework).
- Verifica-se a delegação ao caso de uso e a propagação de falhas.
- Utilizar Fixtures, AssertJ e os verificadores do Mockito.
- Um cenário por teste, com `@DisplayName` e padrão AAA.
