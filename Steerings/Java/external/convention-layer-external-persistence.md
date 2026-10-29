# Convenções de Persistência

A camada de **Persistência** é responsável por armazenar e recuperar dados da aplicação.

Seu objetivo é adaptar o modelo de domínio ao mecanismo de persistência utilizado, mantendo o domínio completamente desacoplado dos detalhes técnicos.

Nesta arquitetura, a implementação da persistência deve estar obrigatoriamente localizada na camada **External**, por representar um detalhe de infraestrutura e uma implementação concreta das portas definidas pela camada Application.

Este guia define um padrão único para organização da persistência e deve ser seguido por todos os projetos que adotarem esta arquitetura.

O uso do **Java Persistence API (JPA)** é obrigatório para implementação da camada de persistência.

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
- Devem estender um repositório do Spring Data.
- Não devem conter regras de negócio.
- Devem trabalhar exclusivamente com Models.
- Não devem ser utilizados diretamente pelas camadas Application ou Presentation.

## ✔ Correto

```java
public interface SampleJpaDao extends JpaRepository<SampleJpaModel, UUID> {

    Optional<SampleJpaModel> findByName(String name);
}
```

## ❌ Incorreto

```java
public class SampleJpaDao { }
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
- Devem utilizar o sufixo `Adapter` (ex.: `SampleRepositoryAdapter`).

## ✔ Correto

```java
@Repository
public class SampleRepositoryAdapter implements SampleRepository {

    private final SampleJpaDao dao;

    public SampleRepositoryAdapter(SampleJpaDao dao) {
        this.dao = dao;
    }

    @Override
    public SampleEntity save(SampleEntity entity) {
        var model = SampleMapper.toModel(entity);

        var saved = dao.save(model);

        return SampleMapper.toDomain(saved);
    }
}
```

## ❌ Incorreto

```java
@Repository
public class SampleRepositoryAdapter implements SampleRepository {

    @Override
    public SampleJpaModel save(
            SampleEntity entity
    ) {
        ...
    }
}
```

---

# Mapper

Mappers são responsáveis exclusivamente pela conversão entre objetos do domínio e modelos de persistência.

As seguintes regras devem ser respeitadas:

- Não devem possuir regras de negócio.
- Devem realizar apenas conversões.
- Devem converter Domain ↔ Model.
- Devem permanecer stateless.
- Devem possuir métodos estáticos quando não houver necessidade de estado.

## ✔ Correto

```java
public final class SampleMapper {

    private SampleMapper() { }

    public static SampleJpaModel toModel(SampleEntity entity) {
        return new SampleJpaModel(
                UUID.fromString(entity.getId().asString()),
                entity.getName().asString(),
                entity.getCreatedAt(),
                entity.getUpdatedAt()
        );
    }

    public static SampleEntity toDomain(SampleJpaModel model) {
        return new SampleEntity(
                model.getId().toString(),
                model.getName(),
                model.getCreatedAt(),
                model.getUpdatedAt()
        );
    }
}
```

## ❌ Incorreto

```java
public class SampleMapper {

    public SampleEntity toDomain(SampleJpaModel model) {
        repository.save(...);
    }
}
```

---

# Model

Models representam exclusivamente a estrutura persistida no banco de dados.

Eles não representam objetos do domínio.

As seguintes regras devem ser respeitadas:

- Devem utilizar anotações JPA.
- Devem representar apenas a estrutura do banco.
- Não devem conter regras de negócio.
- Não devem ser utilizados fora da camada de persistência.
- Não devem ser expostos para Application ou Presentation.
- IDs são do tipo `UUID`; `@Id` sem geração automática (o ID vem do domínio).
- O `createdAt` é do tipo `Instant` **não anulável** e **não** usa `@CreationTimestamp`. O timestamp é um fato definido pelo domínio (a entidade já nasce com `createdAt`), e o Model apenas grava o valor que recebe do mapper — a persistência não gera nem preenche o timestamp. Isso evita o estado transitório inválido de um campo que "nasce `null`" só para ser preenchido depois pelo Hibernate.
- Colunas imutáveis após a criação recebem `updatable = false` (é o caso de `createdAt`).
- O Model expõe os getters, consumidos pelo Mapper no `toModel`/`toDomain`.
- O Model expõe os setters, utilizados para facilitar a criação de testes.

> Quando um domínio específico precisar de `updatedAt`, ele segue o mesmo princípio do `createdAt`: o valor é decidido no domínio (a entidade o atualiza a cada método de negócio que altera o estado) e repassado pelo mapper, sem `@UpdateTimestamp`. A diferença em relação ao `createdAt` é a nulidade: o `updatedAt` nasce `null` (entidade nunca alterada), então a coluna é **anulável** (`nullable = true`) e **não** recebe `updatable = false`, pois precisa ser gravada nas atualizações. Enquanto o domínio não precisar, o Model não declara o campo.

## ✔ Correto

```java
@Entity
@Table(name = "samples")
public class SampleJpaModel {

    @Id
    @Column(nullable = false, updatable = false)
    private UUID id;

    @Column(nullable = false)
    private String name;

    @Column(name = "created_at", nullable = false, updatable = false)
    private Instant createdAt;

    @Column(name = "updated_at")
    private Instant updatedAt;

    public UUID getId() {
        return id;
    }

    public String getName() {
        return name;
    }

    public Instant getCreatedAt() {
        return createdAt;
    }

    public Instant getUpdatedAt() {
        return updatedAt;
    }

    public void setName(String name) {
        this.name = name;
    }
}
```

## ❌ Incorreto

```java
@Entity
public class SampleJpaModel {

    public void activate() {
        ...
    }
}
```

```java
@Entity
public class SampleJpaModel {

    public boolean canExecuteBusinessRule() {
        ...
    }
}
```

---

# Queries

Consultas são operações responsáveis exclusivamente pela recuperação de dados.

Seu objetivo é fornecer as informações necessárias para a camada Application, sem implementar regras de negócio.

As seguintes regras devem ser respeitadas:

- Consultas simples devem ser implementadas diretamente nos DAOs através dos recursos do Spring Data JPA.
- Consultas derivadas pelo nome do método devem ser priorizadas sempre que atenderem ao requisito.
- Consultas mais complexas podem utilizar `@Query`, `Specification` ou `Criteria API`, conforme o padrão adotado pelo projeto.
- Consultas nativas (`nativeQuery`) devem ser utilizadas apenas quando não houver alternativa viável utilizando JPA.
- Toda consulta deve retornar Models da camada de persistência.
- Conversões para objetos do domínio devem ocorrer exclusivamente no Repository Adapter através dos Mappers.
- Consultas não devem implementar regras de negócio.
- Consultas não devem ser executadas diretamente pela camada Application.

## ✔ Correto

```java
public interface SampleJpaDao extends JpaRepository<SampleJpaModel, UUID> {

    Optional<SampleJpaModel> findByName(
            String name
    );

    boolean existsByName(
            String name
    );
}
```

```java
public interface SampleJpaDao extends JpaRepository<SampleJpaModel, UUID> {

    @Query("""
        select s
        from SampleJpaModel s
        where s.createdAt >= :createdAt
    """)
    List<SampleJpaModel> findAllCreatedAfter(
            Instant createdAt
    );
}
```

## ✔ Correto

```java
@Override
public Optional<SampleEntity> findByName(
        SampleNameValueObject name
) {

    return dao.findByName(name.asString())
            .map(SampleMapper::toDomain);
}
```

## ❌ Incorreto

```java
@Service
public class CreateSampleUseCase {

    @PersistenceContext
    private EntityManager entityManager;
}
```

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

# Banco de Dados

A camada de persistência deve abstrair completamente a tecnologia utilizada pelo banco de dados.

As seguintes regras devem ser respeitadas:

- Nenhuma camada fora da External deve conhecer detalhes do banco.
- Consultas devem permanecer encapsuladas nos DAOs.
- Toda comunicação deve ocorrer através dos Repository Adapters.
- O banco de dados nunca deve ser acessado diretamente por casos de uso.

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

# IDs

A geração e persistência de identificadores deve respeitar as regras do domínio.

As seguintes regras devem ser respeitadas:

- IDs do domínio devem permanecer encapsulados em Value Objects.
- Models devem armazenar apenas o valor persistido.
- Conversões entre Value Objects e tipos persistidos devem ocorrer exclusivamente nos Mappers.

## ✔ Correto

```java
public static SampleJpaModel toModel(SampleEntity entity) {

    return new SampleJpaModel(
            UUID.fromString(entity.getId().asString()),
            entity.getName().asString(),
            entity.getCreatedAt(),
            entity.getUpdatedAt()
    );
}
```

## ❌ Incorreto

```java
@Entity
public class SampleJpaModel {

    @Embedded
    private IdValueObject id;
}
```

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
- DAOs devem utilizar Spring Data JPA.
- Repository Adapters implementam exclusivamente portas da camada Application.
- Repository Adapters usam o sufixo `Adapter`.
- Models representam apenas a estrutura persistida.
- Mappers realizam exclusivamente conversões entre Domain e Model.
- Nenhuma regra de negócio deve ser implementada nesta camada.
- Models nunca devem ser expostos para fora da External.
- Toda persistência deve passar pelo fluxo **Repository Adapter → Mapper → DAO → JPA → Banco de Dados**.
