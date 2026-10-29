# Testes da Camada Presentation — Scheduler

Os testes de **Scheduler** têm como objetivo garantir que a tarefa agendada orquestre corretamente o caso de uso correspondente.

Eles seguem os mesmos princípios dos testes de Controller (ver `convention-test-layer-presentation-controller.md`), com uma diferença central: como o scheduler **não tem contrato HTTP**, o teste é **unitário puro** — sem contexto Spring, sem slice web. O gatilho de tempo (`@Scheduled`) e o lock distribuído (`@SchedulerLock`) são responsabilidade do framework e da configuração de bootstrap; o teste não os exercita. O que se testa é o **corpo do método agendado**: que ele delega ao caso de uso, sem regra de negócio.

---

## Estrutura

Os testes devem permanecer organizados conforme o scheduler.

Exemplo:

```text
src/test/kotlin/
└── core/
    └── presentation/
        └── scheduler/
            └── SampleSchedulerTest.kt
```

---

## Convenções

As seguintes regras devem ser respeitadas:

- Cada Scheduler deve possuir sua própria classe de teste.
- O teste é unitário: usar `@ExtendWith(MockKExtension::class)`, declarar o caso de uso com `@MockK` e instanciar o scheduler com `@InjectMockKs`.
- Nunca subir contexto Spring (`@SpringBootTest`, `@WebMvcTest`, etc.): não há nada de Spring para exercitar no corpo da tarefa.
- Não testar o agendamento em si (`@Scheduled`) nem o lock (`@SchedulerLock`) — são responsabilidade do framework/configuração.
- Todo método público (tarefa) do scheduler deve possuir teste.
- Deve ser validado que o caso de uso é invocado, com os argumentos corretos quando houver.
- Utilizar Fixtures para construção de quaisquer DTOs de entrada/saída dos casos de uso.
- Os testes devem utilizar AssertJ e os verificadores do MockK (`verify`).
- Usar backticks para nomes de testes.
- Os testes devem seguir o padrão AAA (Arrange → Act → Assert).

---

## Dependências

Os testes de Scheduler devem utilizar apenas:

- MockK;
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

```kotlin
@ExtendWith(MockKExtension::class)
class SampleSchedulerTest {

    @MockK
    lateinit var processSampleUseCase: ProcessSampleUseCase

    @InjectMockKs
    lateinit var sampleScheduler: SampleScheduler

    @Test
    fun `should invoke use case when processing pending samples`() {
        every { processSampleUseCase.execute() } returns Unit

        sampleScheduler.processPending()

        verify { processSampleUseCase.execute() }
    }

    @Test
    fun `should propagate exception when use case fails`() {
        every { processSampleUseCase.execute() } throws IllegalStateException("boom")

        assertThatThrownBy { sampleScheduler.processPending() }
            .isInstanceOf(IllegalStateException::class.java)
    }
}
```

---

## ❌ Incorreto — subir contexto Spring para testar a tarefa

```kotlin
@SpringBootTest
class SampleSchedulerTest {

    @Autowired
    lateinit var sampleScheduler: SampleScheduler
}
```

---

## ❌ Incorreto — testar o agendador em vez da orquestração

```kotlin
@Test
fun `should run every minute`() {
    // Verificar o intervalo do @Scheduled é testar o framework, não a tarefa.
}
```

---

## Resumo das Convenções

Todo teste de Scheduler deve respeitar os seguintes princípios:

- Cada scheduler possui sua própria classe de teste.
- O teste é unitário puro, com o caso de uso mockado via MockK — sem contexto Spring.
- Não se testa `@Scheduled` nem `@SchedulerLock` (responsabilidade do framework).
- Verifica-se a delegação ao caso de uso e a propagação de falhas.
- Usar `@ExtendWith(MockKExtension::class)`, `@MockK` e `@InjectMockKs`.
- Utilizar Fixtures e AssertJ, com backticks nos nomes dos testes.
- Um cenário por teste, seguindo o padrão AAA.
