# Testes Unitários de Casos de Uso

Os testes de **Casos de Uso** têm como objetivo garantir que o fluxo da aplicação seja executado corretamente.

Eles devem validar exclusivamente o comportamento da camada **Application**, assegurando a correta orquestração entre o domínio e as portas da aplicação.

Casos de uso devem ser testados isoladamente, sem carregar o contexto do Spring e sem acessar componentes reais de infraestrutura.

---

## Estrutura

Os testes devem permanecer organizados conforme o caso de uso.

Exemplo:

```text
src/test/kotlin/
└── core/
    └── application/
        └── usecase/
            └── sample/
                └── CreateSampleUseCaseTest.kt
```

---

## Convenções

As seguintes regras devem ser respeitadas:

- Cada caso de uso deve possuir sua própria classe de teste.
- Dependências devem ser simuladas utilizando MockK.
- O contexto do Spring não deve ser iniciado.
- Apenas a classe concreta do caso de uso deve ser instanciada.
- Todos os fluxos de sucesso e falha devem ser testados.
- O comportamento das portas deve ser validado através de verificações (`verify`).
- Fixtures devem ser utilizadas sempre que possível.
- Cada teste deve validar apenas um cenário.
- Usar `@ExtendWith(MockKExtension::class)`.
- Dependências são declaradas com `@MockK`.
- O caso de uso sob teste com `@InjectMockKs`.

---

## Dependências

Casos de uso devem ser testados utilizando apenas:

- JUnit 5;
- MockK;
- AssertJ;
- Fixtures;
- Objetos do domínio.

Não devem utilizar:

- Spring Boot Test;
- Banco de dados;
- Testcontainers;
- Componentes reais da External.

---

## Cenários obrigatórios

Todo caso de uso deve possuir, no mínimo, testes para:

- **Happy path**: fluxo principal de sucesso.
- **Not found**: entidade não encontrada (quando o use case busca por ID).
- **Access denied**: usuário não é dono do recurso (quando há verificação de ownership).
- **Invalid input**: ID ou campo inválido que falha na criação do value object (quando aplicável).
- Exceções lançadas pelo domínio.
- Exceções lançadas por dependências externas.
- Interação correta com as portas.
- Retorno esperado.

---

## ✔ Correto

```kotlin
@ExtendWith(MockKExtension::class)
class CreateSampleUseCaseTest {

    @MockK
    lateinit var sampleRepository: SampleRepository

    @InjectMockKs
    lateinit var useCase: CreateSampleUseCase

    @Test
    fun `should create a sample successfully`() {
        val sample = SampleFixture.aSample()
        val name = sample.name
        val input = CreateSampleInputDto(name = name.asString())
        val savedSample = SampleFixture.aSample()

        every { sampleRepository.existsByName(name) } returns false
        every { sampleRepository.save(any()) } returns savedSample

        val output = useCase.execute(input)

        assertThat(output.id).isNotNull()
        assertThat(output.name).isEqualTo(name.asString())
        verify { sampleRepository.save(any()) }
    }
}
```

---

## ✔ Correto

```kotlin
@Test
fun `should throw when name already exists`() {
    val sample = SampleFixture.aSample()
    val name = sample.name
    val input = CreateSampleInputDto(name = name.asString())

    every { sampleRepository.existsByName(name) } returns true

    assertThatThrownBy { useCase.execute(input) }.isInstanceOf(SampleNameAlreadyExistsException::class.java)
    verify(exactly = 0) { sampleRepository.save(any()) }
}
```

---

## ❌ Incorreto

```kotlin
@SpringBootTest
class CreateSampleUseCaseTest { }
```

---

## ❌ Incorreto

```kotlin
@Test
fun `should create sample`() {
    val repository = SampleRepositoryAdapter(...)
}
```

---

## ❌ Incorreto

```kotlin
@Test
fun `should create sample`() {
    useCase.execute(input)
}
```

---

## Boas práticas

Sempre que possível:

- Utilizar Fixtures para criação dos objetos.
- Validar o retorno do caso de uso.
- Validar as interações com as dependências.
- Utilizar `confirmVerified(mock)` ao final para garantir que nenhuma interação não verificada ocorreu (equivalente MockK ao `verifyNoMoreInteractions`), quando aplicável.
- Um cenário por teste.
- Evitar múltiplos asserts para comportamentos distintos.
- Nomear os testes de forma descritiva com backticks.
- Manter os testes independentes entre si.

---

## Resumo das Convenções

Todo teste de Caso de Uso deve respeitar os seguintes princípios:

- Cada caso de uso possui sua própria classe de teste.
- Apenas a camada Application deve ser testada.
- Dependências devem ser simuladas utilizando MockK.
- O contexto do Spring não deve ser iniciado.
- Usar `@ExtendWith(MockKExtension::class)`, `@MockK` e `@InjectMockKs`.
- Fixtures devem ser reutilizadas sempre que possível.
- Todo fluxo de sucesso e falha deve ser validado.
- Toda interação com as portas deve ser verificada.
- Cada teste deve validar um único cenário.
- Testes devem ser rápidos, determinísticos e independentes.
