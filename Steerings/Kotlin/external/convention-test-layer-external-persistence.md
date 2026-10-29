# Testes de Persistência

Os testes da camada de **Persistência** têm como objetivo garantir que os componentes responsáveis pelo armazenamento, recuperação e conversão de dados funcionem corretamente.

Nesta arquitetura, a camada de persistência é composta por três componentes principais:

- DAO;
- Mapper;
- Repository Adapter.

Cada componente possui responsabilidades distintas e, consequentemente, estratégias de teste diferentes.

- **Mappers** devem ser testados através de **testes unitários**, por não possuírem dependências externas.
- **DAOs** devem ser testados através de **testes de integração com `@DataJpaTest`**, pois dependem do JPA e de um banco de dados.
- **Repository Adapters** devem ser testados através de **testes unitários com MockK**, validando a orquestração entre DAO e Mapper de forma isolada.

---

## Estrutura

Os testes devem permanecer organizados conforme o componente testado.

Exemplo:

```text
src/test/kotlin/
└── core/
    └── external/
        └── persistence/
            ├── dao/
            │   └── SampleJpaDaoTest.kt
            ├── mapper/
            │   └── SampleMapperTest.kt
            └── repository/
                └── SampleRepositoryAdapterTest.kt
```

---

## Convenções

As seguintes regras devem ser respeitadas:

- Todo DAO deve possuir testes com `@DataJpaTest`.
- Todo Mapper deve possuir testes unitários.
- Todo Repository Adapter deve possuir testes unitários com MockK.
- Cada classe deve testar exclusivamente um componente.
- Fixtures devem ser utilizadas para construção dos objetos.
- Os testes devem utilizar AssertJ.
- Os testes devem ser organizados em cenários de sucesso, corner cases e exceções.

---

## Testes de DAO com @DataJpaTest

Os testes de DAO têm como objetivo validar o comportamento das consultas realizadas pelo Spring Data JPA.

### Convenções

- Devem utilizar `@DataJpaTest`.
- Devem injetar o DAO com `@Autowired`.
- Não devem herdar de `IntegrationTestBase`.
- Podem usar H2 em memória para testes rápidos e isolados. Isso é intencional e **não** contraria a regra "nunca banco em memória" de `convention-test-layer-presentation-controller.md`: aquela regra se aplica exclusivamente aos testes de **integração de controller** (que exercem o fluxo completo e exigem o banco real via Testcontainers). O `@DataJpaTest` valida apenas o mapeamento e as queries do DAO de forma isolada, cenário em que o H2 é suficiente e mais rápido.
- Devem validar métodos derivados do Spring Data.
- Devem validar consultas utilizando `@Query`.
- Não devem testar regras de negócio.
- Use `@BeforeEach` para popular dados de pré-requisito.
- Construa os modelos JPA via método auxiliar privado (`buildSample(...)`) para evitar repetição.
- Constantes para IDs desconhecidos ficam como `private val` no `companion object`.

### Cenários obrigatórios

- Buscar por ID (encontrado e não encontrado).
- FindAll (com resultado e lista vazia).
- Save.
- Delete.
- Exists.
- Consultas customizadas.

### ✔ Correto

```kotlin
@DataJpaTest
class SampleJpaDaoTest {

    @Autowired
    private lateinit var sampleJpaDao: SampleJpaDao

    @BeforeEach
    fun setUp() {
        val sample = SampleFixture.aSample()

        val buildedSample = buildSample(sample)

        sampleJpaDao.save(buildedSample)
    }

    @Test
    fun `should return empty when sample id does not exist`() {
        val found = sampleJpaDao.findById(UNKNOWN_SAMPLE_ID)

        assertThat(found).isEmpty()
    }

    @Test
    fun `should save and retrieve a sample`() {
        val sample = SampleFixture.aSample()
        val model = buildSample(sample)

        sampleJpaDao.save(model)

        val found = sampleJpaDao.findById(model.id)
        assertThat(found).isPresent()
    }

    private fun buildSample(sample: SampleEntity): SampleJpaModel =
        SampleJpaModel(
            id = UUID.fromString(sample.id.asString()),
            name = sample.name.asString(),
            createdAt = sample.createdAt
        )

    companion object {
        private val UNKNOWN_SAMPLE_ID: UUID = UUID.fromString("00000000-0000-0000-0000-000000000002")
    }
}
```

---

## Testes Unitários de Mapper

Os testes de Mapper têm como objetivo garantir a correta conversão entre Domain e Persistência.

### Convenções

- Sem anotações de extensão — são classes Kotlin puras.
- Não devem utilizar Spring.
- Não devem utilizar banco de dados.
- Devem validar conversão Domain → Model (`toModel`).
- Devem validar conversão Model → Domain (`toDomain`).
- Devem validar todos os atributos.
- Devem validar conversão de `UUID` ↔ `String`.
- Um teste por direção de mapeamento.

### ✔ Correto

```kotlin
class SampleMapperTest {

    @Test
    fun `should map a SampleEntity to a SampleJpaModel correctly`() {
        val sample = SampleFixture.aSample()
        val sampleId = sample.id
        val createdAt = sample.createdAt

        val model = SampleMapper.toModel(sample)

        assertThat(model.id.toString()).isEqualTo(sampleId.asString())
        assertThat(model.name).isEqualTo(sample.name.asString())
        assertThat(model.createdAt).isEqualTo(createdAt)
        assertThat(model.updatedAt).isEqualTo(sample.updatedAt)
    }

    @Test
    fun `should map a SampleJpaModel to a SampleEntity correctly`() {
        val sample = SampleFixture.aSample()
        val model = SampleMapper.toModel(sample)

        val entity = SampleMapper.toDomain(model)

        assertThat(entity.id.asString()).isEqualTo(model.id.toString())
        assertThat(entity.name.asString()).isEqualTo(model.name)
        assertThat(entity.createdAt).isEqualTo(model.createdAt)
        assertThat(entity.updatedAt).isEqualTo(model.updatedAt)
    }
}
```

---

## Testes Unitários de Repository Adapter

Os testes de Repository Adapter têm como objetivo garantir que o adaptador orquestre corretamente as conversões (Mapper) e delegações (DAO).

### Convenções

- Devem utilizar `@ExtendWith(MockKExtension::class)`.
- Devem simular o DAO utilizando `@MockK`.
- Devem instanciar o Repository Adapter com `@InjectMockKs`.
- Não devem utilizar contexto Spring.
- Não devem utilizar banco de dados.
- Devem validar operações de persistência (delegação ao DAO).
- Devem validar que os resultados retornados foram corretamente convertidos.
- Não devem testar regras de negócio.

### Cenários obrigatórios

- Save.
- FindById (presente e vazio/null).
- FindAll (com resultado e lista vazia).
- Exists.
- Delete.

### ✔ Correto

```kotlin
@ExtendWith(MockKExtension::class)
class SampleRepositoryAdapterTest {

    @MockK
    lateinit var jpaRepository: SampleJpaDao

    @InjectMockKs
    lateinit var adapter: SampleRepositoryAdapter

    @Test
    fun `should save a sample and return the persisted domain entity`() {
        val sample = SampleFixture.aSample()
        val sampleId = sample.id
        val model = buildSampleModel()

        every { jpaRepository.save(any()) } returns model

        val result = adapter.save(sample)

        assertThat(result.id.asString()).isEqualTo(sampleId.asString())
        verify { jpaRepository.save(any()) }
    }

    private fun buildSampleModel(): SampleJpaModel =
        SampleJpaModel(
            id = UUID.fromString(SampleFixture.DEFAULT_ID),
            name = SampleFixture.DEFAULT_NAME,
            createdAt = SampleFixture.DEFAULT_CREATED_AT
        )
}
```

---

## Boas práticas

Sempre que possível:

- Utilizar Fixtures.
- Validar todos os atributos convertidos.
- Validar os cenários positivos e negativos.
- Um cenário por teste.
- Evitar dependência entre testes.
- Validar apenas a responsabilidade do componente em teste.

---

## Resumo das Convenções

Toda implementação da camada de Persistência deve respeitar os seguintes princípios:

- Todo DAO deve possuir testes com `@DataJpaTest`.
- Todo Mapper deve possuir testes unitários (classes Kotlin puras).
- Todo Repository Adapter deve possuir testes unitários com MockK.
- DAOs devem utilizar `@DataJpaTest` e `@Autowired`.
- Repository Adapters devem utilizar `@ExtendWith(MockKExtension::class)`, `@MockK` e `@InjectMockKs`.
- Mappers devem ser testados isoladamente.
- Utilizar Fixtures para criação dos objetos.
- Utilizar AssertJ para todas as validações.
- Usar backticks para nomes de testes.
- Os testes devem ser rápidos, determinísticos e independentes.
