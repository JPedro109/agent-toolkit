# Testes da Camada Presentation — Controller (HTTP)

# Testes Unitários de Controllers

Os testes de **Controllers** têm como objetivo garantir que a camada Presentation esteja corretamente integrada ao contrato HTTP da aplicação.

Eles devem validar exclusivamente o comportamento do controller, incluindo requisições, respostas, validações, serialização, desserialização e códigos HTTP.

Controllers não devem ser testados juntamente com a lógica de negócio, que pertence aos Casos de Uso.

---

## Estrutura

Os testes devem permanecer organizados conforme o controller.

Exemplo:

```text
src/test/kotlin/
└── core/
    └── presentation/
        └── controller/
            └── SampleControllerTest.kt
```

---

## Convenções

As seguintes regras devem ser respeitadas:

- Cada Controller deve possuir sua própria classe de teste.
- Os testes devem utilizar `@WebMvcTest(XxxController::class)`.
- O slice carrega o Controller sob teste e, via `@Import`, a configuração de segurança (`SecurityConfig`) e o `GlobalExceptionHandler` — necessários para exercitar autenticação/autorização (401/403) e a tradução de exceções no próprio teste de controller.
- Casos de Uso são simulados com `@MockkBean` (SpringMockK). A porta `Token` também é mockada com `@MockkBean`, pois o `SecurityConfig`/filtro de autenticação depende dela para montar o contexto de segurança.
- Endpoints protegidos simulam o usuário autenticado com `@WithJwtTokenMock` no método de teste; cenários sem a anotação exercitam o retorno 401.
- Todos os endpoints devem possuir testes.
- Deve ser validado o código HTTP retornado.
- Deve ser validado o payload retornado.
- Deve ser validado o comportamento das validações Bean Validation.
- Deve ser validado o tratamento de erros.
- Utilizar Fixtures para construção dos DTOs de saída dos Casos de Uso.
- Usar backticks para nomes de testes.
- Os testes devem utilizar AssertJ quando houver validações fora do MockMvc.
- Os testes devem seguir o padrão AAA (Arrange → Act → Assert).
- Uma `inner class` por endpoint, anotada com `@Nested`, com `@DisplayName` indicando o método HTTP e o path.
- Cada classe nested tem seu próprio método privado `perform(...)` que encapsula a chamada MockMvc para aquele endpoint.

---

## Dependências

Os testes de Controller devem utilizar apenas:

- Spring MVC Test (`MockMvc`);
- MockK / SpringMockK;
- Jackson;
- Fixtures.

Não devem utilizar:

- Banco de dados;
- Repositories;
- Adaptadores da External;
- Casos de uso reais.

---

## Cenários obrigatórios

Todo Controller deve possuir, no mínimo, testes para:

- Requisição válida.
- Payload inválido.
- Campos obrigatórios ausentes.
- Erros de Bean Validation.
- Exceções de negócio.
- Recursos inexistentes.
- Códigos HTTP esperados.
- Serialização e desserialização do payload.

---

## ✔ Correto

```kotlin
@WebMvcTest(SampleController::class)
@Import(SecurityConfig::class, GlobalExceptionHandler::class)
class SampleControllerTest {

    @Autowired
    private lateinit var mockMvc: MockMvc

    @MockkBean
    private lateinit var createSampleUseCase: CreateSampleUseCase

    @MockkBean
    private lateinit var token: Token

    @Nested
    @DisplayName("POST /api/v1/samples")
    inner class CreateSample {

        @Test
        fun `should return 201 with sample data when input is valid`() {
            val sample = SampleFixture.aSample()
            val name = sample.name
            val output = SampleOutputDto(id = sample.id.asString(), name = name.asString())

            every { createSampleUseCase.execute(any()) } returns output

            perform(name = name.asString())
                .andExpect(status().isCreated)
                .andExpect(jsonPath("$.name").value(name.asString()))
        }

        private fun perform(name: String): ResultActions {
            val requestBody = """{"name": "$name"}"""
            return mockMvc.perform(
                post("/api/v1/samples")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content(requestBody)
            )
        }
    }
}
```

---

## ❌ Incorreto

```kotlin
@SpringBootTest
class SampleControllerTest { }
```

---

## ❌ Incorreto

```kotlin
@Test
fun `should create sample`() {
    val controller = SampleController(...)
}
```

---

## Boas práticas

Sempre que possível:

- Validar o corpo da resposta utilizando `jsonPath`.
- Validar o `Content-Type` retornado.
- Validar os códigos HTTP esperados.
- Utilizar Fixtures para os DTOs de saída.
- Validar que o Caso de Uso foi chamado corretamente.
- Validar que o Caso de Uso não foi chamado em cenários inválidos.
- Manter um único cenário por teste.

---

# Testes de Integração de Controller

Os testes de integração de controller têm como objetivo validar o comportamento completo da aplicação através de seu contrato HTTP.

Eles devem exercitar todo o fluxo da requisição, desde o recebimento pelo Controller até a persistência dos dados, utilizando as implementações reais da aplicação.

---

## Estrutura

Todos os testes devem permanecer organizados na pasta `integration`.

Exemplo:

```text
src/test/kotlin/
└── integration/
    ├── common/
    │   ├── abstracts/IntegrationTestBase.kt
    │   ├── container/PostgresContainerConfig.kt
    │   └── sql/SqlCreateSeed.kt
    └── SampleIntegrationTest.kt
```

Todos os testes de integração dos controllers devem herdar da classe abstrata `IntegrationTestBase`:

```kotlin
@SpringBootTest(classes = [SampleApplication::class])
@AutoConfigureMockMvc
@ActiveProfiles("integration-test")
@Import(PostgresContainerConfig::class)
abstract class IntegrationTestBase {

    @Autowired
    protected lateinit var mockMvc: MockMvc

    @Autowired
    protected lateinit var objectMapper: ObjectMapper
}
```

Quaisquer configurações de Testcontainer devem ser criados em `integration/common/container/`, uma classe por dependência:

```kotlin
@TestConfiguration(proxyBeanMethods = false)
class PostgresContainerConfig {

    @Bean
    @ServiceConnection
    fun postgresContainer(): PostgreSQLContainer<*> =
        PostgreSQLContainer(DockerImageName.parse("postgres:15-alpine"))
            .withDatabaseName("sample_test")
            .withUsername("test")
            .withPassword("test")
}
```

Quaisquer configurações de seeds para testes devem ficar em `integration/common/sql/`:

```kotlin
@Target(AnnotationTarget.FUNCTION, AnnotationTarget.CLASS)
@Retention(AnnotationRetention.RUNTIME)
@SqlGroup(
    Sql(
        scripts = ["classpath:/sql/insert-sample.sql"],
        executionPhase = Sql.ExecutionPhase.BEFORE_TEST_METHOD
    ),
    Sql(
        scripts = ["classpath:/sql/cleanup.sql"],
        executionPhase = Sql.ExecutionPhase.AFTER_TEST_METHOD
    )
)
annotation class SqlCreateSeed
```

---

## Convenções

As seguintes regras devem ser respeitadas:

- Cada Controller deve possuir uma classe de teste com o sufixo `IntegrationTest`.
- Toda classe deve estender `IntegrationTestBase`.
- Nunca configurar manualmente `MockMvc`, `SpringBootTest`, `Testcontainers` ou perfil de execução.
- Testes de integração de controller devem utilizar Testcontainers — nunca banco de dados em memória. (Essa regra vale para a integração de controller; os testes de DAO com `@DataJpaTest`, que podem usar H2 em memória, são tratados em `convention-test-layer-external-persistence.md`.)
- Nunca utilizar mocks da aplicação.
- Todos os objetos válidos utilizados durante os testes devem ser construídos utilizando Fixtures.
- Para cenários de dados inválidos, os campos válidos devem ser extraídos de Fixtures normalmente. Apenas o campo inválido deve ser declarado manualmente.
- Cada endpoint deve possuir uma `inner class` (`@Nested`).
- Cada classe `@Nested` deve possuir um `@DisplayName` indicando o método HTTP e o path.
- Cada classe `@Nested` deve possuir um método privado `perform(...)`.
- Cada teste deve declarar explicitamente as anotações necessárias (`@SqlCreateSeed`, `@WithJwtTokenMock`).
- Nenhum teste deve depender da execução de outro.
- Usar backticks para nomes de testes.
- Os testes devem seguir o padrão AAA (Arrange → Act → Assert).

---

## Testcontainers

Todos os recursos externos utilizados pelos testes devem ser executados através de **Testcontainers**.

Não é permitido utilizar bancos de dados em memória para testes de integração de controller.

Toda nova dependência externa adicionada ao projeto deve possuir sua própria configuração de Testcontainer em `integration/common/container/`.

### Convenções

- Cada dependência externa deve possuir sua própria classe de configuração.
- Nunca declarar múltiplos containers na mesma classe.
- As configurações devem utilizar `@TestConfiguration`.
- Os containers devem ser registrados através de `@Bean`.
- Sempre utilizar `@ServiceConnection` quando suportado.

### ✔ Correto

```kotlin
@TestConfiguration(proxyBeanMethods = false)
class PostgresContainerConfig {

    @Bean
    @ServiceConnection
    fun postgresContainer(): PostgreSQLContainer<*> =
        PostgreSQLContainer(DockerImageName.parse("postgres:15-alpine"))
}
```

---

## Cenários obrigatórios

Todo endpoint deve possuir, no mínimo, testes para:

- Fluxo de sucesso.
- Payload inválido.
- Recursos inexistentes.
- Erros de autenticação (401), quando aplicável.
- Erros de autorização (403), quando aplicável.
- Validação das regras de negócio.

---

## ✔ Correto

```kotlin
@DisplayName("Sample Integration Tests")
class SampleIntegrationTest : IntegrationTestBase() {

    @Nested
    @DisplayName("POST /api/v1/samples")
    inner class CreateSample {

        @Test
        @SqlCreateSeed
        @WithJwtTokenMock
        fun `should return 201 with sample data when input is valid`() {
            val name = "Sample Name"

            perform(name)
                .andExpect(status().isCreated)
                .andExpect(jsonPath("$.id").isNotEmpty)
                .andExpect(jsonPath("$.name").value(name))
        }

        @Test
        fun `should return 401 when no token is provided`() {
            perform("Sample Name")
                .andExpect(status().isUnauthorized)
        }

        @Test
        @SqlCreateSeed
        @WithJwtTokenMock
        fun `should return 400 when sample name is blank`() {
            perform("")
                .andExpect(status().isBadRequest)
        }

        private fun perform(name: String): ResultActions {
            val requestBody = """{"name": "$name"}"""
            return mockMvc.perform(
                post("/api/v1/samples")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content(requestBody)
            )
        }
    }
}
```

---

## ❌ Incorreto

```kotlin
@SpringBootTest
class SampleIntegrationTest { }
```

---

## ❌ Incorreto

```kotlin
@MockkBean
private lateinit var sampleRepository: SampleRepository
```

---

## Boas práticas

Sempre que possível:

- Criar uma `inner class` `@Nested` para cada endpoint.
- Centralizar a chamada HTTP no método `perform(...)`.
- Declarar explicitamente todas as anotações necessárias para cada cenário.
- Utilizar Fixtures para criação dos objetos válidos.
- Para cenários inválidos, extrair os campos válidos da Fixture e declarar manualmente apenas o campo inválido.
- Validar o código HTTP retornado.
- Validar o corpo da resposta.
- Manter um único cenário por teste.

---

## Resumo das Convenções

### Testes unitários de Controller

- Uma classe de teste por Controller.
- Utilizar `@WebMvcTest`.
- Casos de Uso devem ser simulados com `@MockkBean`.
- Apenas a camada Presentation deve ser testada.
- Uma `inner class` `@Nested` por endpoint.
- Um método privado `perform(...)` por endpoint.
- Utilizar Fixtures para os DTOs de saída.
- Usar backticks para nomes de testes.
- Os testes devem ser rápidos, determinísticos e independentes.

### Testes de integração de Controller

- Uma classe `IntegrationTest` por Controller.
- Herdar obrigatoriamente de `IntegrationTestBase`.
- Utilizar Testcontainers e banco de dados real.
- Nunca utilizar mocks.
- Uma `inner class` `@Nested` por endpoint.
- Um método privado `perform(...)` por endpoint.
- Utilizar Fixtures para todos os cenários válidos.
- Declarar explicitamente `@SqlCreateSeed`, `@WithJwtTokenMock` e demais anotações necessárias.
- Usar backticks para nomes de testes.
- Validar o contrato HTTP e o comportamento completo da aplicação.
- Os testes devem ser independentes, determinísticos e reproduzíveis.
