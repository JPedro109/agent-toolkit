# Convenções de testes

# Layout

```text
src/test/java/
├── core/
│   ├── application/              # Testes unitários dos casos de uso (Mockito, sem contexto Spring)
│   ├── domain/                   # Testes unitários de entidades, value objects e serviços de domínio
│   ├── external/                 # Testes de integração dos adaptadores de infraestrutura (repositories, mappers, segurança, integrações)
│   └── presentation/            # Testes unitários dos controllers (MockMvc/WebMvcTest)
├── integration/                  # Testes de integração dos controllers
│   ├── common/
│   │   ├── abstracts/
│   │   │   └── IntegrationTestBase.java
│   │   ├── container/
│   │   │   └── PostgresContainerConfig.java
│   │   └── sql/
│   │       └── SqlCreateSeed.java
│   └── SampleIntegrationTest.java
└── shared/
    ├── fixture/
    │   └── SampleFixture.java
    └── security/
        └── WithJwtTokenMock.java
```

Todos os testes de integração dos controllers devem herdar da classe abstrata em `src/test/java/integration/common/abstracts`, que deve ser esta:

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

Quaisquer configurações de test container devem ser criados em src/test/java/integration/common/container, uma classe por dependência, exemplo:

```java
@SuppressWarnings("resource")
@TestConfiguration(proxyBeanMethods = false)
public class PostgresContainerConfig {

    @Bean
    @ServiceConnection
    public PostgreSQLContainer<?> postgresContainer() {
        return new PostgreSQLContainer<>(DockerImageName.parse("postgres:15-alpine"))
                .withDatabaseName("sample_test")
                .withUsername("test")
                .withPassword("test");
    }
}
```

Quaisquer configurações de seeds para testes devem ficar em `src/test/java/integration/common/sql`, exemplo:

```java
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@SqlGroup({
        @Sql(
                scripts = {
                        "classpath:/sql/insert-sample.sql"
                },
                executionPhase = Sql.ExecutionPhase.BEFORE_TEST_METHOD
        ),
        @Sql(
                scripts = "classpath:/sql/cleanup.sql",
                executionPhase = Sql.ExecutionPhase.AFTER_TEST_METHOD
        )
})
public @interface SqlCreateSeed { }
```


# Geral

- Use ou crie fixtures existentes fixtures para construir dados de teste. Nunca instancie entidades de domínio inline dentro dos testes. Nunca use valores aleatórios (`UUID.randomUUID()`, `Math.random()`, etc.) — sempre fixture.
- Todo método de teste deve ter `@DisplayName` com uma frase descritiva em inglês começando por `"Should [resultado esperado]"`. Quando o cenário depende de uma condição, inclua a cláusula `when [condição]`: `"Should [resultado esperado] when [condição]"`. A cláusula `when` pode ser omitida apenas quando o teste não tem condição a destacar (um happy path incondicional).
  - Exemplos: `"Should create sample when data is valid"`, `"Should throw when sample is not found"`, `"Should return 400 when name is blank"`, `"Should create sample"`.
- O nome do método é o `@DisplayName` em camelCase: `shouldCreateSampleWhenDataIsValid`, `shouldThrowWhenSampleNotFound`.
- Use `assertThat` do AssertJ para asserções e `assertThatThrownBy` para verificar exceções.
- Verifique interações com `verify(mock).method(...)` e `verify(mock, never()).method(...)`.
- Testes devem ser ordenados: sucesso (happy path) primeiro, corner cases depois, exceções/erros por último.
- Use `@BeforeEach` e `@AfterEach` quando necessário para setup e teardown compartilhados.
- Siga o padrão **AAA (Arrange → Act → Assert)**, separando cada etapa com uma linha em branco, nunca utilize comentários para separação das etapas.
  - Dentro do `Arrange`, separe a criação de variáveis dos stubs `when(...)` com uma linha em branco:

```java
var sample = SampleFixture.aSample();
var name = sample.getName();
var input = new CreateSampleInputDto(name.asString());
var savedSample = SampleFixture.aSample();

when(sampleRepository.existsByName(name)).thenReturn(false);
when(sampleRepository.save(any())).thenReturn(savedSample);

var output = useCase.execute(input);

assertThat(output.name()).isEqualTo(name.asString());
assertThat(output.id()).isNotNull();
verify(sampleRepository).save(any());
```

- Sempre declare o fixture primeiro e depois extraia cada campo que for utilizar em variáveis separadas:

```java
var sample = SampleFixture.aSample();
var name = sample.getName();
```

- Cubra o máximo de cenários possível. Sempre siga a estrutura de layout definida para cada tipo de teste.

# Fixtures

Fixtures são responsáveis por centralizar a criação de objetos utilizados durante os testes.

Seu objetivo é evitar duplicação de código, facilitar a leitura dos testes e manter um único ponto de manutenção para dados de teste.

Todos os fixtures compartilhados devem permanecer no pacote `shared.fixture`.

As seguintes regras devem ser respeitadas:

- Devem ser classes utilitárias.
- Não devem possuir estado.
- Devem possuir construtor privado.
- Devem disponibilizar métodos estáticos para criação dos objetos.
- Devem fornecer valores padrão válidos para todos os atributos obrigatórios.
- Devem permitir a criação rápida de objetos válidos para diferentes cenários de teste.
- Devem ser reutilizados por todos os testes da aplicação sempre que possível.

Estrutura recomendada:

```text
shared/
└── fixture/
    ├── SampleFixture.java
```

## ✔ Correto

```java
public final class SampleFixture {
    public static final String DEFAULT_ID = "00000000-0000-0000-0000-000000000001";
    public static final String DEFAULT_NAME = "Sample";
    public static final Instant DEFAULT_CREATED_AT = Instant.parse("2024-01-01T00:00:00Z");

    private SampleFixture() { }

    public static SampleEntity aSample() {
        return new SampleEntity(
                DEFAULT_ID,
                DEFAULT_NAME,
                DEFAULT_CREATED_AT,
                null
        );
    }
}
```

O `createdAt` da fixture é um valor fixo e determinístico — nunca `Instant.now()`. Isso mantém os testes reproduzíveis e permite asserir que o `createdAt` sobrevive ao round-trip de persistência. O `updatedAt` começa `null` (entidade recém-criada, nunca alterada); quando o teste precisar de uma entidade já atualizada, passe um `Instant` fixo nesse argumento.

## ✔ Utilização

```java
@Test
void shouldCreateSample() {
    var sample = SampleFixture.aSample();
    ...
}
```

## ❌ Incorreto

```java
@Test
void shouldCreateSample() {
    var sample = new SampleEntity(
            "00000000-0000-0000-0000-000000000001",
            "Sample"
    );
    ...
}
```

# WireMock

Toda integração HTTP com sistemas externos deve ser testada utilizando **WireMock**.

O objetivo é simular o comportamento de APIs externas de forma determinística, eliminando dependências de ambientes externos durante a execução dos testes.

As seguintes regras devem ser respeitadas:

- Nunca realizar chamadas HTTP para ambientes reais durante os testes.
- Toda API externa deve possuir uma classe responsável por centralizar seus stubs.
- Os testes nunca devem utilizar `stubFor(...)` diretamente.
- Cada método da classe de stub deve representar um cenário de teste.
- Os stubs devem validar método HTTP, URL, parâmetros, headers e corpo da requisição sempre que aplicável.
- Cada teste deve configurar apenas os stubs necessários para o cenário em execução.
- Devem existir cenários de sucesso, erro, timeout e indisponibilidade do serviço.
- Nunca utilizar Mockito para simular clientes HTTP concretos.

Estrutura recomendada:

```text
src/test/java/
└── integration/
    └── common/
        ├── container/
        │   └── WireMockConfig.java
        └── wiremock/
            ├── SampleApiStub.java
```

Cada integração HTTP deve possuir sua própria classe de stubs.

## ✔ Correto

```java
public final class SampleApiStub {

    private SampleApiStub() { }

    public static void sampleExists(String id) {
        stubFor(get(urlEqualTo("/samples/" + id))
                .willReturn(okJson("""
                        {
                            "exists": true
                        }
                        """)));
    }

    public static void sampleDoesNotExist(String id) {
        stubFor(get(urlEqualTo("/samples/" + id))
                .willReturn(okJson("""
                        {
                            "exists": false
                        }
                        """)));
    }

    public static void serviceUnavailable() {
        stubFor(get(urlPathMatching("/samples/.*"))
                .willReturn(serverError()));
    }

    public static void timeout() {
        stubFor(get(urlPathMatching("/samples/.*"))
                .willReturn(aResponse()
                        .withFixedDelay(5000)
                        .withStatus(200)));
    }

}
```

## ✔ Utilização

```java
@Test
@DisplayName("Should create sample when it does not exist")
void shouldCreateSampleWhenItDoesNotExist() throws Exception {
    SampleApiStub.sampleDoesNotExist("1");

    perform().andExpect(status().isCreated());
}
```

```java
@Test
@DisplayName("Should return conflict when sample already exists")
void shouldReturnConflictWhenSampleAlreadyExists() throws Exception {
    SampleApiStub.sampleExists("1");

    perform().andExpect(status().isConflict());
}
```

```java
@Test
@DisplayName("Should return service unavailable when external service is unavailable")
void shouldReturnServiceUnavailableWhenExternalServiceIsUnavailable() throws Exception {
    SampleApiStub.serviceUnavailable();

    perform().andExpect(status().isServiceUnavailable());
}
```

## ❌ Incorreto

```java
@Test
void shouldCreateSample() throws Exception {

    stubFor(get(urlEqualTo("/samples/1"))
            .willReturn(okJson("""
                    {
                        "exists": false
                    }
                    """)));

    perform();
}
```

O uso de `stubFor(...)` diretamente nos testes dificulta a reutilização, aumenta a duplicação de código e espalha detalhes de implementação do WireMock pelos testes.

## ❌ Incorreto

```java
when(sampleApiClient.exists(any()))
        .thenReturn(false);
```

- Clientes HTTP concretos nunca devem ser simulados com Mockito. 
- O comportamento da integração deve ser exercitado utilizando WireMock.
