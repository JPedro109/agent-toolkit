# Testes Unitários de Controllers
# Testes da Camada Presentation — Controller (HTTP)

Os testes de **Controllers** têm como objetivo garantir que a camada Presentation esteja corretamente integrada ao contrato HTTP da aplicação.

Eles devem validar exclusivamente o comportamento do controller, incluindo requisições, respostas, validações, serialização, desserialização e códigos HTTP.

Controllers não devem ser testados juntamente com a lógica de negócio, que pertence aos Casos de Uso.

---

## Estrutura

Os testes devem permanecer organizados conforme o controller.

Exemplo:

```text
src/test/java/
└── core/
    └── presentation/
        └── controller/
            └── SampleControllerTest.java
```

---

## Convenções

As seguintes regras devem ser respeitadas:

- Cada Controller deve possuir sua própria classe de teste.
- Os testes devem utilizar `@WebMvcTest`.
- O slice carrega o Controller sob teste e, via `@Import`, a configuração de segurança (`SecurityConfig`), o `GlobalExceptionHandler` e o `AuthenticatedUserResolver` — necessários para exercitar autenticação/autorização (401/403) e a tradução de exceções no próprio teste de controller.
- Casos de Uso são simulados com `@MockitoBean`. A porta `Token` também é mockada com `@MockitoBean`, pois o `SecurityConfig`/filtro de autenticação depende dela para montar o contexto de segurança.
- Endpoints protegidos simulam o usuário autenticado com `@WithJwtTokenMock` no método de teste; cenários sem a anotação exercitam o retorno 401.
- Todos os endpoints devem possuir testes.
- Deve ser validado o código HTTP retornado.
- Deve ser validado o payload retornado.
- Deve ser validado o comportamento das validações Bean Validation.
- Deve ser validado o tratamento de erros.
- Utilizar Fixtures para construção dos DTOs de saída dos Casos de Uso.
- Todo método deve possuir `@DisplayName`.
- Os testes devem utilizar AssertJ quando houver validações fora do MockMvc.
- Os testes devem seguir o padrão AAA (Arrange → Act → Assert).
- Uma `@Nested` class por endpoint, com `@DisplayName` indicando o método HTTP e o path (ex: `"POST /api/v1/samples"`).
- Cada classe nested tem seu próprio método privado `perform(...)` que encapsula a chamada MockMvc para aquele endpoint.

---

## Dependências

Os testes de Controller devem utilizar apenas:

- Spring MVC Test (`MockMvc`);
- Mockito;
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

```java
@WebMvcTest(SampleController.class)
@Import({SecurityConfig.class, GlobalExceptionHandler.class, AuthenticatedUserResolver.class})
class SampleControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockitoBean
    private CreateSampleUseCase createSampleUseCase;

    @MockitoBean
    private Token token;

    @Nested
    @DisplayName("POST /api/v1/samples")
    class CreateSample {

        @Test
        @DisplayName("Should return 201 when request is valid")
        void shouldReturn201WhenRequestIsValid() throws Exception {
            var sample = SampleFixture.aSample();
            var sampleName = sample.getName().asString();
            var output = buildSampleOutput(sample);

            when(createSampleUseCase.execute(any()))
                    .thenReturn(output);

            var result = perform(sampleName);

            result.andExpect(status().isCreated());
            verify(createSampleUseCase).execute(any());
        }

        @Test
        @DisplayName("Should return 400 when name is blank")
        void shouldReturn400WhenNameIsBlank() throws Exception {
            var invalidSampleName = "";

            var result = perform(invalidSampleName);

            result.andExpect(status().isBadRequest());
            verify(createSampleUseCase, never()).execute(any());
        }

        private ResultActions perform(String name) throws Exception {
            var requestBody = """
                    {"name": "%s"}
                    """.formatted(name);

            return mockMvc.perform(post("/api/v1/samples")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content(requestBody));
        }
    }

    private SampleOutputDto buildSampleOutput(SampleEntity sample) {
        return new SampleOutputDto(
                sample.getId().asString(),
                sample.getName().asString()
        );
    }
}
```

---

## ✔ Correto

```java
@Test
@DisplayName("Should return 400 when name is blank")
void shouldReturn400WhenNameIsBlank() throws Exception {
    var invalidSampleName = "";

    var result = perform(invalidSampleName);

    result.andExpect(status().isBadRequest());
    verify(createSampleUseCase, never()).execute(any());
}
```

---

## ❌ Incorreto

```java
@SpringBootTest
class SampleControllerTest { }
```

---

## ❌ Incorreto

```java
@MockitoBean
private SampleRepository repository;
```

---

## ❌ Incorreto

```java
@Test
void shouldCreateSample() {
    var controller = new SampleController(...);
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

## Resumo das Convenções

Todo teste de Controller deve respeitar os seguintes princípios:

- Cada Controller possui sua própria classe de teste.
- Utilizar `@WebMvcTest`.
- Casos de Uso devem ser simulados utilizando `@MockitoBean`.
- Apenas a camada Presentation deve ser testada.
- Todos os endpoints devem ser cobertos.
- Deve ser validado o código HTTP e o payload retornado.
- Deve ser validado o Bean Validation.
- Deve ser validado o tratamento de erros.
- Utilizar Fixtures para os DTOs de saída.
- Todo método deve possuir `@DisplayName`.
- Os testes devem ser rápidos, determinísticos e independentes.

# Testes de Integração de Controllers

Os testes integração de controllers têm como objetivo validar o comportamento completo da aplicação através de seu contrato HTTP.

Eles devem exercitar todo o fluxo da requisição, desde o recebimento pelo Controller até a persistência dos dados, utilizando as implementações reais da aplicação.

Os testes devem validar a integração entre todas as camadas da arquitetura e garantir que o comportamento observado pelo cliente seja o esperado.

---

## Estrutura

Todos os testes devem permanecer organizados na pasta `integration`.

Exemplo:

```text
src/test/java/
└── integration/
    ├── common/
    │   ├── abstracts/
    │   │   └── IntegrationTestBase.java
    │   ├── container/
    │   │   └── PostgresContainerConfig.java
    │   └── sql/
    │       └── SqlCreateSeed.java
    └── sample/
        └── SampleIntegrationTest.java
```

> Cada dependência externa adicionada ao projeto ganha sua própria classe de container em `integration/common/container/` (ex.: `RedisContainerConfig`, `RabbitMqContainerConfig`), e cada seed adicional sua própria anotação em `integration/common/sql/` (ex.: `SqlUpdateSeed`). A estrutura acima mostra apenas o mínimo (Postgres + seed de criação); acrescente os demais conforme a necessidade real do projeto.

---

## Convenções

As seguintes regras devem ser respeitadas:

- Cada Controller deve possuir uma classe de teste com o sufixo `IntegrationTest`.
- Toda classe deve estender `IntegrationTestBase`.
- Nunca configurar manualmente `MockMvc`, `SpringBootTest`, `Testcontainers` ou perfil de execução.
- Todos os testes **de integração de controllers** devem utilizar Testcontainers.
- Nunca utilizar banco de dados em memória **nos testes de integração de controllers**. Esta regra se aplica a este tipo de teste, que sobe a aplicação completa e deve exercitar o Postgres real. Os testes de DAO (`@DataJpaTest`), regidos por `convention-test-layer-external-persistence.md`, usam intencionalmente o banco embarcado (H2) e não estão sujeitos a esta regra.
- Nunca utilizar mocks da aplicação.
- Todos os objetos válidos utilizados durante os testes devem ser construídos utilizando Fixtures.
- Para cenários de dados inválidos, os campos válidos devem ser extraídos de Fixtures normalmente. Apenas o campo inválido deve ser declarado manualmente com o valor que representa a falha. Nunca utilizar uma Fixture que retorne um objeto inválido completo.
- Nunca instanciar entidades do domínio manualmente quando existir uma Fixture correspondente.
- Cada endpoint deve possuir uma classe interna (`@Nested`).
- Cada classe `@Nested` deve representar um único endpoint.
- Cada classe `@Nested` deve possuir um método privado `perform(...)` responsável por encapsular a chamada ao `MockMvc`.
- Cada teste deve declarar explicitamente as anotações necessárias, como `@SqlCreateSeed` e `@WithJwtTokenMock`.
- Nenhum teste deve depender da execução de outro.
- Todo método deve possuir `@DisplayName`.
- Os testes devem seguir o padrão AAA (Arrange → Act → Assert).

---

## Classe Base

Todos os testes devem herdar da classe base da aplicação.

### ✔ Correto

```java
@SpringBootTest(classes = SampleApplication.class)
@AutoConfigureMockMvc
@ActiveProfiles("integration-test")
@Import(PostgresContainerConfig.class)
public abstract class IntegrationTestBase {

    @Autowired
    protected MockMvc mockMvc;
}
```

---

## Testcontainers

Todos os recursos externos utilizados pelos testes devem ser executados através de **Testcontainers**.

Nos testes de integração de controllers, não é permitido utilizar bancos de dados em memória, serviços mockados ou configurações específicas para substituir dependências reais da aplicação. (Os testes de DAO com `@DataJpaTest` são a exceção documentada em `convention-test-layer-external-persistence.md` e usam banco embarcado.)

Toda nova dependência externa adicionada ao projeto deve possuir sua própria configuração de Testcontainer.

Essas configurações devem permanecer centralizadas no pacote `integration.common.container`.

### Estrutura

```text
src/test/java/
└── integration/
    └── common/
        └── container/
            └── PostgresContainerConfig.java
```

Cada nova dependência externa (Redis, RabbitMQ, LocalStack, etc.) acrescenta sua própria classe de container neste pacote, uma por arquivo.

### Convenções

As seguintes regras devem ser respeitadas:

- Cada dependência externa deve possuir sua própria classe de configuração.
- Nunca declarar múltiplos containers na mesma classe.
- As configurações devem utilizar `@TestConfiguration`.
- Os containers devem ser registrados através de `@Bean`.
- Sempre utilizar `@ServiceConnection` quando suportado pelo Spring Boot.
- Todas as configurações devem permanecer em `integration.common.container`.

### ✔ Correto

```java
@TestConfiguration(proxyBeanMethods = false)
public class PostgresContainerConfig {

    @Bean
    @ServiceConnection
    PostgreSQLContainer<?> postgresContainer() {
        return new PostgreSQLContainer<>(
                DockerImageName.parse("postgres:15-alpine")
        );
    }
}
```

### ✔ Correto

```java
@TestConfiguration(proxyBeanMethods = false)
public class RedisContainerConfig {

    @Bean
    @ServiceConnection
    RedisContainer redisContainer() {
        return new RedisContainer(
                DockerImageName.parse("redis:7-alpine")
        );
    }
}
```

### ❌ Incorreto

```java
@TestConfiguration
public class ContainersConfig {

    @Bean
    PostgreSQLContainer<?> postgres() { ... }

    @Bean
    RedisContainer redis() { ... }

    @Bean
    RabbitMQContainer rabbit() { ... }
}
```

---

## Organização dos Endpoints

Cada endpoint deve ser representado por uma classe `@Nested`.

A classe deve possuir um `@DisplayName` indicando o método HTTP e o caminho da requisição.

### ✔ Correto

```java
@DisplayName("Sample Integration Tests")
class SampleIntegrationTest extends IntegrationTestBase {
    @Nested
    @DisplayName("POST /api/v1/samples")
    class CreateSample { }

    @Nested
    @DisplayName("GET /api/v1/samples/{id}")
    class FindSampleById { }
}
```

---

## Método perform

Cada endpoint deve encapsular a chamada ao `MockMvc` em um método privado denominado `perform(...)`.

Esse método deve conter exclusivamente a construção da requisição HTTP.

### ✔ Correto

```java
private ResultActions perform(String name) throws Exception {
    var requestBody = """
            {"name": "%s"}
            """.formatted(name);

    return mockMvc.perform(
            post("/api/v1/samples")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content(requestBody)
    );
}
```

---

## Cenários obrigatórios

Todo endpoint deve possuir, no mínimo, testes para:

- Fluxo de sucesso.
- Payload inválido.
- Recursos inexistentes.
- Erros de autenticação, quando aplicável.
- Erros de autorização, quando aplicável.
- Validação das regras de negócio.
- Tratamento de exceções.

---

### ✔ Correto

```java
@DisplayName("Sample Integration Tests")
class SampleIntegrationTest extends IntegrationTestBase {
    @Nested
    @DisplayName("POST /api/v1/samples")
    class CreateSample {

        @Test
        @DisplayName("Should return 201 when request is valid")
        @SqlCreateSeed
        @WithJwtTokenMock
        void shouldReturn201WhenRequestIsValid() throws Exception {
            var sample = SampleFixture.aSample();
            var name = sample.getName();

            var result = perform(name.asString());

            result.andExpect(status().isCreated());
        }

        @Test
        @DisplayName("Should return 400 when name is blank")
        @SqlCreateSeed
        @WithJwtTokenMock
        void shouldReturn400WhenNameIsBlank() throws Exception {
            var invalidName = "";

            var result = perform(invalidName);

            result.andExpect(status().isBadRequest());
        }

        @Test
        @DisplayName("Should return 401 when no token is provided")
        void shouldReturn401WhenNoTokenIsProvided() throws Exception {
            var sample = SampleFixture.aSample();
            var name = sample.getName();

            var result = perform(name.asString());

            result.andExpect(status().isUnauthorized());
        }

        private ResultActions perform(String name) throws Exception {
            var requestBody = """
                    {"name": "%s"}
                    """.formatted(name);

            return mockMvc.perform(
                    post("/api/v1/samples")
                            .contentType(MediaType.APPLICATION_JSON)
                            .content(requestBody)
            );
        }
    }
}
```

---

### ❌ Incorreto

```java
@SpringBootTest
class SampleIntegrationTest { }
```

---

### ❌ Incorreto

```java
@MockBean
private SampleRepository repository;
```

---

### ❌ Incorreto

```java
@Test
void shouldReturn400WhenNameIsBlank() throws Exception {
    perform("Sample Name");
}
```

Neste caso, o teste afirma validar o cenário de nome em branco (`400`), mas envia um nome válido. O correto é declarar explicitamente o valor inválido que representa a falha (ex.: `var invalidSampleName = "";`) e passá-lo ao `perform(...)`, de modo que o corpo enviado reflita exatamente o cenário sob teste.

---

## Boas práticas

Sempre que possível:

- Criar uma classe `@Nested` para cada endpoint.
- Centralizar a chamada HTTP no método `perform(...)`.
- Declarar explicitamente todas as anotações necessárias para cada cenário.
- Utilizar Fixtures para criação dos objetos válidos.
- Nunca criar entidades ou payloads manualmente quando existir uma Fixture correspondente para cenários válidos.
- Para cenários inválidos, extrair os campos válidos da Fixture e declarar manualmente apenas o campo inválido que representa a falha.
- Validar o código HTTP retornado.
- Validar o corpo da resposta.
- Utilizar Seeds para preparação dos cenários.
- Manter um único cenário por teste.

---

## Resumo das Convenções

Todo teste deve respeitar os seguintes princípios:

- Uma classe `IntegrationTest` por Controller.
- Herdar obrigatoriamente de `IntegrationTestBase`.
- Utilizar Testcontainers e banco de dados real.
- Toda nova dependência externa deve possuir seu próprio Testcontainer em `integration.common.container`.
- Nunca utilizar mocks.
- Uma classe `@Nested` por endpoint.
- Um método privado `perform(...)` por endpoint.
- Utilizar Fixtures para todos os cenários válidos.
- Para cenários inválidos, extrair os campos válidos da Fixture e declarar manualmente apenas o campo inválido.
- Declarar explicitamente `@SqlCreateSeed`, `@WithJwtTokenMock` e demais anotações necessárias.
- Todo método deve possuir `@DisplayName`.
- Validar o contrato HTTP e o comportamento completo da aplicação.
- Os testes devem ser independentes, determinísticos e reproduzíveis.
