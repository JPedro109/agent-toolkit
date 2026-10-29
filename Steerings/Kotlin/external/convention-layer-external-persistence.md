# Convenções de Persistência

A camada de **Persistência** é responsável por armazenar e recuperar dados da aplicação.

Seu objetivo é adaptar o modelo de domínio ao mecanismo de persistência utilizado, mantendo o domínio completamente desacoplado dos detalhes técnicos.

Nesta arquitetura, a implementação da persistência deve estar obrigatoriamente localizada na camada **External**, por representar um detalhe de infraestrutura e uma implementação concreta das portas definidas pela camada Application.

O uso do **Java Persistence API (JPA)** via Spring Data JPA é obrigatório para implementação da camada de persistência.

---

# Estrutura

A organização recomendada é:

```text
external/
└── persistence/
    ├── dao/
    ├── mapper/
    ├── model/
    └── repository/
```

Cada componente possui uma responsabilidade específica e não deve assumir responsabilidades pertencentes aos demais.

---

# DAO

DAOs representam a implementação de acesso aos dados utilizando o mecanismo de persistência adotado.

Nesta arquitetura, todos os DAOs devem utilizar **Spring Data JPA**.

As seguintes regras devem ser respeitadas:

- Devem ser interfaces.
- Devem estender `JpaRepository<Model, UUID>`.
- Não devem conter regras de negócio.
- Devem trabalhar exclusivamente com Models.
- Não devem ser utilizados diretamente pelas camadas Application ou Presentation.
- Declaram apenas os métodos de query adicionais necessários. Métodos padrão do `JpaRepository` são usados diretamente.

## ✔ Correto

```kotlin
interface SampleJpaDao : JpaRepository<SampleJpaModel, UUID> {
    fun findByName(name: String): SampleJpaModel?
    fun existsByName(name: String): Boolean
}
```

## ❌ Incorreto

```kotlin
class SampleJpaDao { }
```

---

# Repository Adapter

Repository Adapters representam os adaptadores responsáveis por implementar as portas de persistência definidas na camada Application.

Seu papel é converter objetos do domínio em modelos de persistência e delegar as operações ao DAO.

As seguintes regras devem ser respeitadas:

- Devem implementar exclusivamente interfaces da camada Application.
- Devem depender apenas do DAO e dos Mappers.
- Devem trabalhar com Entidades do domínio.
- Não devem implementar regras de negócio.
- Não devem expor Models para outras camadas.
- Devem utilizar o sufixo `Adapter` (ex: `SampleRepositoryAdapter`).
- Anotados com `@Repository`.

## ✔ Correto

```kotlin
@Repository
class SampleRepositoryAdapter(
    private val jpaRepository: SampleJpaDao
) : SampleRepository {

    override fun save(sample: SampleEntity): SampleEntity {
        val model = SampleMapper.toModel(sample)

        val saved = jpaRepository.save(model)

        return SampleMapper.toDomain(saved)
    }

    override fun findById(id: IdValueObject): SampleEntity? {
        val parsedId = UUID.fromString(id.asString())

        return jpaRepository.findById(parsedId).map { SampleMapper.toDomain(it) }.orElse(null)
    }
}
```

## ❌ Incorreto

```kotlin
@Repository
class SampleRepositoryAdapter : SampleRepository {

    override fun save(sample: SampleEntity): SampleJpaModel {
        ...
    }
}
```

---

# Mapper

Mappers são responsáveis exclusivamente pela conversão entre objetos do domínio e modelos de persistência.

As seguintes regras devem ser respeitadas:

- Implementados como `object` Kotlin (equivalente a classes estáticas sem estado).
- Não devem possuir regras de negócio.
- Devem realizar apenas conversões.
- `toModel(entity)` — converte domínio para JPA.
- `toDomain(model)` — converte JPA para domínio.
- Nunca adicionam lógica de negócio ou validação.

## ✔ Correto

```kotlin
object SampleMapper {

    fun toModel(entity: SampleEntity): SampleJpaModel =
        SampleJpaModel(
            id = UUID.fromString(entity.id.asString()),
            name = entity.name.asString(),
            createdAt = entity.createdAt,
            updatedAt = entity.updatedAt
        )

    fun toDomain(model: SampleJpaModel): SampleEntity =
        SampleEntity(
            id = model.id.toString(),
            name = model.name,
            createdAt = model.createdAt,
            updatedAt = model.updatedAt
        )
}
```

## ❌ Incorreto

```kotlin
object SampleMapper {

    fun toDomain(model: SampleJpaModel): SampleEntity {
        repository.save(...)
    }
}
```

---

# Model

Models representam exclusivamente a estrutura persistida no banco de dados.

Eles não representam objetos do domínio.

As seguintes regras devem ser respeitadas:

- Devem utilizar anotações JPA (`@Entity`, `@Table`, `@Column`).
- Devem representar apenas a estrutura do banco.
- Não devem conter regras de negócio.
- Não devem ser utilizados fora da camada de persistência.
- Não devem ser expostos para Application ou Presentation.
- Campos `UUID` para IDs (não `String`) — a conversão é feita nos mappers.
- `@Id` sem geração automática — o ID vem do domínio.
- O `createdAt` é do tipo `Instant` **não anulável** e **não** usa `@CreationTimestamp`. O timestamp é um fato definido pelo domínio (a entidade já nasce com `createdAt`), e o Model apenas grava o valor que recebe do mapper — a persistência não gera nem preenche o timestamp. Isso evita o estado transitório inválido de um campo anulável que "nasce `null`" só para ser preenchido depois pelo Hibernate.
- Colunas imutáveis após criação recebem `updatable = false` (é o caso de `createdAt`).
- O plugin `kotlin-jpa` gera automaticamente o construtor sem argumentos exigido pelo JPA.
- Use um único construtor primário com todos os campos.

> Quando um domínio específico precisar de `updatedAt`, ele segue o mesmo princípio do `createdAt`: o valor é decidido no domínio (a entidade o atualiza a cada método de negócio que altera o estado) e repassado pelo mapper, sem `@UpdateTimestamp`. A diferença em relação ao `createdAt` é a nulidade: o `updatedAt` nasce `null` (entidade nunca alterada), então a coluna é **anulável** (`nullable = true`) e **não** recebe `updatable = false`, pois precisa ser gravada nas atualizações. Enquanto o domínio não precisar, o Model não declara o campo.

## ✔ Correto

```kotlin
@Entity
@Table(name = "samples")
class SampleJpaModel(
    @Id
    @Column(nullable = false, updatable = false)
    val id: UUID,

    @Column(nullable = false)
    var name: String,

    @Column(name = "created_at", nullable = false, updatable = false)
    val createdAt: Instant,

    @Column(name = "updated_at")
    var updatedAt: Instant? = null
)
```

## ❌ Incorreto

```kotlin
@Entity
class SampleJpaModel {
    fun activate() { ... }
}
```

---

# Queries

Consultas são operações responsáveis exclusivamente pela recuperação de dados.

As seguintes regras devem ser respeitadas:

- Consultas simples devem ser implementadas diretamente nos DAOs através dos recursos do Spring Data JPA.
- Consultas derivadas pelo nome do método devem ser priorizadas sempre que atenderem ao requisito.
- Consultas mais complexas podem utilizar `@Query`.
- Toda consulta deve retornar Models da camada de persistência.
- Conversões para objetos do domínio devem ocorrer exclusivamente no Repository Adapter através dos Mappers.
- Consultas não devem implementar regras de negócio.
- Consultas não devem ser executadas diretamente pela camada Application.

## ✔ Correto

```kotlin
interface SampleJpaDao : JpaRepository<SampleJpaModel, UUID> {

    fun findByName(name: String): SampleJpaModel?

    fun existsByName(name: String): Boolean

    @Query("""
        SELECT s FROM SampleJpaModel s
        WHERE s.createdAt >= :createdAt
    """)
    fun findAllCreatedAfter(createdAt: Instant): List<SampleJpaModel>
}
```

## ❌ Incorreto

```kotlin
@Service
class CreateSampleUseCase(
    @PersistenceContext
    private val entityManager: EntityManager
)
```

---

# Fluxo de Persistência

Toda operação de persistência deve seguir o fluxo abaixo:

```text
Application
      ↓
Repository Adapter
      ↓
Mapper
      ↓
DAO
      ↓
JPA
      ↓
Banco de Dados
```

O fluxo inverso deve seguir o mesmo princípio:

```text
Banco de Dados
      ↓
JPA
      ↓
DAO
      ↓
Mapper
      ↓
Repository Adapter
      ↓
Application
```

---

# IDs

A geração e persistência de identificadores deve respeitar as regras do domínio.

As seguintes regras devem ser respeitadas:

- IDs do domínio devem permanecer encapsulados em Value Objects (`IdValueObject`).
- IDs do domínio são sempre strings UUID na fronteira do domínio.
- Models devem armazenar IDs como objetos `UUID`.
- Conversões entre `String` e `UUID` devem ocorrer exclusivamente nos Mappers com `UUID.fromString(...)` / `.toString()`.

## ✔ Correto

```kotlin
fun toModel(entity: SampleEntity): SampleJpaModel =
    SampleJpaModel(
        id = UUID.fromString(entity.id.asString()),
        name = entity.name.asString()
    )
```

## ❌ Incorreto

```kotlin
@Entity
class SampleJpaModel(
    @Embedded
    var id: IdValueObject
)
```

---

# Migrações

Toda alteração estrutural do banco de dados deve ser realizada através do **Flyway**.

As seguintes regras devem ser respeitadas:

- Alterações manuais no banco não são permitidas.
- Scripts devem ser versionados seguindo o padrão `V{n}__{descricao}.sql`.
- Migrações devem ser executadas automaticamente durante o processo de implantação.
- Cada migração deve representar uma única alteração estrutural.
- Nunca use `ddl-auto: create/update`.

---

# Dependências

A camada de Persistência pode depender de:

- Domain;
- Application;
- Spring Data JPA;
- JPA;
- Hibernate.

Não é permitido depender de:

- Presentation.

---

# Resumo das Convenções

Toda implementação da camada de Persistência deve respeitar os seguintes princípios:

- O uso de **JPA** é obrigatório.
- DAOs devem utilizar Spring Data JPA e estender `JpaRepository`.
- Repository Adapters implementam exclusivamente portas da camada Application.
- Repository Adapters usam sufixo `Adapter`.
- Models representam apenas a estrutura persistida.
- Mappers são `object` Kotlin e realizam exclusivamente conversões entre Domain e Model.
- Nenhuma regra de negócio deve ser implementada nesta camada.
- Models nunca devem ser expostos para fora da camada External.
- Toda persistência deve passar pelo fluxo **Repository Adapter → Mapper → DAO → JPA → Banco de Dados**.
