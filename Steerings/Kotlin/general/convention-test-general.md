# Convenções de Testes

# Layout

```text
src/test/kotlin/<pacote-base-do-projeto>/
├── core/
│   ├── application/usecase/       # Testes unitários dos casos de uso (MockK, sem contexto Spring)
│   │   ├── sample/
│   ├── domain/                    # Testes unitários de entidades e value objects
│   │   ├── sample/
│   ├── external/                  # Testes unitários de adaptadores e mappers
│   │   ├── persistence/
│   │   │   ├── dao/
│   │   │   ├── mapper/
│   │   │   └── repository/
│   │   └── security/
│   └── presentation/
│       └── controller/            # Testes unitários dos controllers (@WebMvcTest)
│           └── SampleControllerTest.kt
├── integration/                   # Testes de integração dos controllers (Testcontainers PostgreSQL)
│   ├── common/
│   │   ├── abstracts/IntegrationTestBase.kt
│   │   ├── container/PostgresContainerConfig.kt
│   │   └── sql/SqlCreateSeed.kt
│   ├── SampleIntegrationTest.kt
└── shared/
    ├── fixture/                   # Fixtures compartilhados
    │   └── SampleFixture.kt
    └── security/
        ├── WithJwtTokenMock.kt
        └── factory/WithMockJwtTokenSecurityContextFactory.kt
```

O scaffolding de integração — a classe base `IntegrationTestBase`, a configuração de container `PostgresContainerConfig` e a anotação de seed `SqlCreateSeed` — tem seu código de referência completo em `convention-test-layer-presentation-controller.md` (seção "Testes de Integração de Controller"). Toda classe de teste de integração herda de `IntegrationTestBase`, e cada dependência externa possui sua própria configuração de container em `integration/common/container/`.

O scaffolding de segurança de teste reside em `shared/security/`: `WithJwtTokenMock` é uma anotação de teste que injeta um usuário autenticado no `SecurityContext`, e `WithMockJwtTokenSecurityContextFactory` é a `WithSecurityContextFactory` que a materializa. Os testes de integração de endpoints protegidos anotam o método com `@WithJwtTokenMock` para simular a autenticação sem gerar um token real.

# Geral

- Use ou crie fixtures existentes para construir dados de teste. Nunca instancie entidades de domínio inline dentro dos testes. Nunca use valores aleatórios (`UUID.randomUUID()`, `Math.random()`, etc.) — sempre fixture.
- Todo método de teste **deve** usar a sintaxe de backticks do Kotlin com uma frase descritiva em inglês no formato `` `should [resultado esperado] when [condição]` ``. Nunca use camelCase para nomear métodos de teste.

```kotlin
// CORRETO
@Test
fun `should return 201 with sample data when input is valid`() { }

// INCORRETO
@Test
fun shouldReturn201WithSampleDataWhenInputIsValid() { }
```

- Use `assertThat` do AssertJ para asserções e `assertThatThrownBy` para verificar exceções.
- Verifique interações com `verify { mock.method(...) }` e `verify(exactly = 0) { mock.method(...) }`.
- Testes devem ser ordenados: sucesso (happy path) primeiro, corner cases depois, exceções/erros por último.
- Use `@BeforeEach` e `@AfterEach` quando necessário para setup e teardown compartilhados.
- Siga o padrão **AAA (Arrange → Act → Assert)**, separando cada etapa com uma linha em branco, nunca utilize comentários para separação das etapas.
- Dentro do `Arrange`, separe a criação de variáveis dos stubs `every { ... }` com uma linha em branco:

```kotlin
val sample = SampleFixture.aSample()
val name = sample.name
val input = CreateSampleInputDto(name = name.asString())

every { sampleRepository.save(any()) } returns sample

val output = useCase.execute(input)

assertThat(output.id).isNotNull()
assertThat(output.name).isEqualTo(name.asString())
verify { sampleRepository.save(any()) }
```

- Sempre declare o fixture primeiro e depois extraia **cada campo que for utilizado** em variáveis separadas — nunca acesse propriedades do fixture diretamente no meio do código do teste:

```kotlin
// CORRETO
val sample = SampleFixture.aSample()
val name = sample.name
val sampleId = sample.id

// INCORRETO — acesso inline sem extração
every { sampleRepository.findById(sample.id) } returns sample
assertThat(output.name).isEqualTo(sample.name.asString())
```

- Cubra o máximo de cenários possível. Sempre siga a estrutura de layout definida para cada tipo de teste.

---

# Fixtures

Fixtures são responsáveis por centralizar a criação de objetos utilizados durante os testes.

Seu objetivo é evitar duplicação de código, facilitar a leitura dos testes e manter um único ponto de manutenção para dados de teste.

Todos os fixtures compartilhados devem permanecer no pacote `shared/fixture`.

As seguintes regras devem ser respeitadas:

- Devem ser implementados como `object` Kotlin.
- Não devem possuir estado.
- Devem disponibilizar métodos para criação dos objetos.
- Devem fornecer valores padrão válidos para todos os atributos obrigatórios.
- Devem permitir a criação rápida de objetos válidos para diferentes cenários de teste.
- Devem ser reutilizados por todos os testes da aplicação sempre que possível.
- Valores padrão reutilizados devem ser expostos como constantes do `object`: `const val` para primitivos e `String`; `val` para tipos que não admitem `const` (ex.: `Instant`). Constantes consumidas pelos testes (como `DEFAULT_ID`) são públicas; as de uso interno do fixture, `private`.

## ✔ Correto

```kotlin
object SampleFixture {
    const val DEFAULT_ID = "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
    const val DEFAULT_NAME = "Sample Name"
    val DEFAULT_CREATED_AT: Instant = Instant.parse("2024-01-01T00:00:00Z")

    fun aSample(): SampleEntity =
        SampleEntity(
            id = DEFAULT_ID,
            name = DEFAULT_NAME,
            createdAt = DEFAULT_CREATED_AT
        )
}
```

O `createdAt` da fixture é um valor fixo e determinístico — nunca `Instant.now()`. Isso mantém os testes reproduzíveis e permite asserir que o `createdAt` sobrevive ao round-trip de persistência. O `updatedAt` usa o padrão `null` (entidade recém-criada, nunca alterada); quando o teste precisar de uma entidade já atualizada, passe um `Instant` fixo via `updatedAt = ...`.

## ✔ Utilização

```kotlin
@Test
fun `should create sample successfully`() {
    val sample = SampleFixture.aSample()
    ...
}
```

## ❌ Incorreto

```kotlin
@Test
fun `should create sample successfully`() {
    val sample = SampleEntity(
        id = "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
        name = "Sample Name"
    )
    ...
}
```

---

# WireMock

Toda integração HTTP com sistemas externos deve ser testada utilizando **WireMock**.

As seguintes regras devem ser respeitadas:

- Nunca realizar chamadas HTTP para ambientes reais durante os testes.
- Toda API externa deve possuir uma classe responsável por centralizar seus stubs.
- Os testes nunca devem utilizar `stubFor(...)` diretamente.
- Cada método da classe de stub deve representar um cenário de teste.
- Os stubs devem validar método HTTP, URL, parâmetros, headers e corpo da requisição sempre que aplicável.
- Cada teste deve configurar apenas os stubs necessários para o cenário em execução.
- Devem existir cenários de sucesso, erro, timeout e indisponibilidade do serviço.
- Nunca utilizar MockK para simular clientes HTTP concretos.

Estrutura recomendada:

```text
src/test/kotlin/
└── integration/
    └── common/
        ├── container/
        │   └── WireMockConfig.kt
        └── wiremock/
            └── SampleApiStub.kt
```

> A configuração do WireMock (`WireMockConfig`) reside em `integration/common/container/`, junto das demais configurações de infraestrutura de teste; os stubs ficam em `integration/common/wiremock/`.

## ✔ Correto

```kotlin
object SampleApiStub {

    fun sampleExists(id: String) {
        stubFor(
            get(urlEqualTo("/samples/$id"))
                .willReturn(okJson("""{"exists": true}"""))
        )
    }

    fun sampleDoesNotExist(id: String) {
        stubFor(
            get(urlEqualTo("/samples/$id"))
                .willReturn(okJson("""{"exists": false}"""))
        )
    }

    fun serviceUnavailable() {
        stubFor(
            get(urlPathMatching("/samples/.*"))
                .willReturn(serverError())
        )
    }

    fun timeout() {
        stubFor(
            get(urlPathMatching("/samples/.*"))
                .willReturn(
                    aResponse()
                        .withFixedDelay(5000)
                        .withStatus(200)
                )
        )
    }
}
```

## ❌ Incorreto

```kotlin
@Test
fun `should create sample`() {
    stubFor(get(urlEqualTo("/samples/1"))
        .willReturn(okJson("""{"exists": false}""")))
    ...
}
```

## ❌ Incorreto

```kotlin
every { sampleApiClient.exists(any()) } returns false
```
