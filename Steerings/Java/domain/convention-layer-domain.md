# Convenções da Camada Domain

A camada **Domain** representa o núcleo da aplicação e concentra exclusivamente as regras de negócio.

Seu objetivo é modelar o domínio de forma independente de frameworks, bancos de dados, protocolos de comunicação ou qualquer outro detalhe técnico.

Toda implementação nesta camada deve priorizar:

- Encapsulamento das regras de negócio;
- Baixo acoplamento;
- Alta coesão;
- Imutabilidade sempre que possível;
- Independência de detalhes técnicos.

A camada Domain **não pode depender** das camadas de Application, External ou Presentation.

---

# Estrutura

A organização da camada Domain deve refletir os conceitos do domínio.

A estrutura recomendada é:

```text
domain/
├── common/
│   ├── abstracts/
│   │   └── Entity.java
│   ├── exception/
│   │   └── DomainException.java
│   └── valueobject/
│       └── IdValueObject.java
└── sample/
    ├── SampleEntity.java
    └── valueobject/
        ├── SampleNameValueObject.java
        └── exception/
            └── InvalidSampleNameException.java
```

As seguintes regras devem ser respeitadas:

- Cada agregado deve possuir seu próprio pacote.
- Cada entidade deve possuir um único arquivo.
- Cada Value Object deve possuir um único arquivo.
- Exceções específicas devem permanecer dentro do agregado correspondente.
- Componentes compartilhados devem permanecer em `domain/common`.

---

# Entidades

Entidades representam objetos do domínio que possuem identidade própria durante todo o seu ciclo de vida.

Toda entidade deve herdar da classe abstrata `Entity`.

A classe base é responsável por centralizar comportamentos comuns entre todas as entidades.

Ela deve fornecer, quando aplicável:

- Identificador;
- Data de criação;
- Método `validateOrThrow()`;
- Igualdade baseada na identidade.

A classe base **não** modela `updatedAt`: atualização não é um comportamento universal — entidades imutáveis após a criação (eventos, registros de log, lançamentos) nunca são alteradas. Quando o agregado tiver ciclo de atualização, a própria entidade declara seu `updatedAt` (`Instant`), como mostrado no exemplo adiante.

## ✔ Correto

```java
public class SampleEntity extends Entity {

    private SampleNameValueObject name;
    private Instant updatedAt;

    public SampleEntity(
            String id,
            String name
    ) {
        this(id, name, null, null);
    }

    public SampleEntity(
            String id,
            String name,
            Instant createdAt,
            Instant updatedAt
    ) {
        super(id, createdAt);

        var nameResult = SampleNameValueObject.of(name);

        var results = List.of(nameResult);
        validateOrThrow(results);

        this.name = nameResult.getValue();
        this.updatedAt = updatedAt;
    }

    public SampleNameValueObject getName() {
        return name;
    }

    public Instant getUpdatedAt() {
        return updatedAt;
    }

    public void updateName(String name) {
        this.name = SampleNameValueObject.of(name).getValueOrThrow();
        this.updatedAt = Instant.now();
    }
}
```

## ✔ Classe base

```java
public abstract class Entity {

    private final IdValueObject id;
    private final Instant createdAt;

    public Entity(String id, Instant createdAt) {
        this.id = IdValueObject.of(id).getValueOrThrow();
        this.createdAt = createdAt != null ? createdAt : Instant.now();
    }

    public IdValueObject getId() {
        return id;
    }

    public Instant getCreatedAt() {
        return createdAt;
    }

    protected void validateOrThrow(List<Result<?>> results) {
        var errors = results.stream()
                .filter(Result::isFail)
                .map(e -> (DomainException) e.getError())
                .toList();

        if (!errors.isEmpty()) {
            throw DomainException.with(errors);
        }
    }
}
```

As seguintes regras devem ser respeitadas:

- Toda entidade deve herdar de `Entity`.
- Entidades representam comportamento, não apenas dados.
- Não devem possuir setters públicos.
- Alterações de estado devem ocorrer através de métodos de negócio.
- Toda entidade deve validar seu estado durante sua construção.
- Nenhuma entidade pode permanecer inválida após sua criação.

O construtor recebe o `createdAt` (`Instant`): quando a entidade é criada pela primeira vez, passa-se `null` (via o construtor de criação) e a classe base assume `Instant.now()`; quando a entidade é reconstruída a partir da persistência, o mapper repassa o `createdAt` armazenado, preservando o valor original. O `createdAt` é, portanto, um fato do domínio — definido aqui, não gerado pela infraestrutura.

Quando o agregado tiver ciclo de atualização, a entidade declara seu próprio `updatedAt` (`Instant`) — ele não pertence à classe base. O `updatedAt` começa `null` numa entidade recém-criada (nunca foi alterada) e passa a `Instant.now()` a cada método de negócio que muda o estado. O construtor de reconstrução recebe o `updatedAt` armazenado (que pode ser `null` ou um `Instant`), enquanto o construtor de criação passa `null`. Assim como o `createdAt`, é um fato do domínio.


## ✔ Correto

```java
sample.updateName(name);
```

## ❌ Incorreto

```java
sample.setName(name);
```

## ❌ Incorreto

```java
@Entity
@Table(name = "sample")
public class SampleEntity { }
```

---

# Objetos Base

Objetos reutilizáveis entre agregados devem permanecer em `domain/common`.

Estrutura recomendada:

```text
common/
├── abstracts/
│   └── Entity.java
├── exception/
│   └── DomainException.java
└── valueobject/
    └── IdValueObject.java
```

As abstrações base recomendadas são:

- Entity
- DomainException
- IdValueObject

Novas abstrações somente devem ser adicionadas quando representarem conceitos reutilizáveis do domínio.

---

# Value Objects

Value Objects representam conceitos definidos exclusivamente pelo seu valor.

Todo Value Object deve encapsular apenas uma informação do domínio.

As seguintes regras devem ser respeitadas:

- Devem ser imutáveis.
- Devem validar seu próprio estado.
- Não devem possuir setters.
- Não devem possuir identidade.
- Devem implementar igualdade baseada no valor.
- Devem ser responsáveis pelas regras referentes ao valor encapsulado.
- Devem conter o método de conversão para o tipo primitivo.
- Devem conter os métodos `equals` e `hashCode` customizados.

Sua criação deve ocorrer através de um método estático retornando `Result<T>`.

## ✔ Correto

```java
public final class SampleNameValueObject {

    private static final int MAX_LENGTH = 255;

    private final String value;

    private SampleNameValueObject(String value) {
        this.value = value;
    }

    public static Result<SampleNameValueObject> of(String value) {
        if (value == null || value.isBlank() || value.length() > MAX_LENGTH) {
            return Result.fail(new InvalidSampleNameException());
        }

        return Result.success(new SampleNameValueObject(value));
    }


    public String asString() {
        return value;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) {
            return true;
        }

        if (!(o instanceof SampleNameValueObject other)) {
            return false;
        }

        return asString().equals(other.asString());
    }

    @Override
    public int hashCode() {
        return asString().hashCode();
    }
}
```

## ❌ Incorreto

```java
public class SampleNameValueObject {

    private String value;

    public void setValue(String value) {
        this.value = value;
    }

}
```

---

# Result

A criação de objetos do domínio deve ocorrer, preferencialmente, através do objeto `Result`.

O `Result<T>` permite representar sucesso ou falha sem depender do lançamento imediato de exceções.

Ele deve ser utilizado principalmente na criação de Value Objects.

Toda exceção retornada deverá ser filha do tipo `RuntimeException`.

## ✔ Correto

```java
var result = SampleNameValueObject.of(value);

if (result.isFail()) {
    throw result.getError();
}

var name = result.getValue();
```

## ✔ Correto

```java
var name = SampleNameValueObject.of(value).getValueOrThrow();
```

## ✔ Correto

```java
var result = SampleNameValueObject.of(value);
```

## ❌ Incorreto

```java
var name = new SampleNameValueObject(value);
```

---

# Validações

Toda regra de validação pertencente ao domínio deve permanecer na camada Domain.

As seguintes regras devem ser respeitadas:

- Entidades devem proteger suas invariantes.
- Value Objects devem validar seu próprio estado.
- Nenhuma entidade pode existir em estado inválido.
- Regras de negócio não devem ser implementadas fora da camada Domain.

---

# Exceções

Exceções da camada Domain representam violações das regras de negócio.

Todas devem herdar da exceção base `DomainException`.

As seguintes regras devem ser respeitadas:

- Devem representar apenas regras de negócio.
- Devem possuir nomes descritivos.
- Devem possuir uma mensagem customizada interna.
- Não devem representar detalhes técnicos.
- Não devem depender de infraestrutura.

## ✔ Correto

```java
public class InvalidSampleNameException extends DomainException {

    public InvalidSampleNameException() {
        super("Sample name must not be blank and must have at most 255 characters");
    }
}
```

## ❌ Incorreto

```java
public class SQLException extends RuntimeException { }
```

---

# Serviços de Domínio

Serviços de domínio devem existir apenas quando um comportamento não pertencer naturalmente a uma única entidade ou Value Object.

As seguintes regras devem ser respeitadas:

- Devem representar regras de negócio.
- Devem ser stateless.
- Não devem depender de infraestrutura.
- Devem operar exclusivamente sobre objetos do domínio.

## ✔ Correto

```java
public class SampleValidationService {

    public void validate(
            SampleEntity sample
    ) {
        ...
    }
}
```

## ❌ Incorreto

```java
public class SampleService {
    private final SampleRepository repository;
}
```

---

# Dependências

A camada Domain deve permanecer completamente independente de detalhes técnicos.

Não é permitido utilizar:

- Spring Framework;
- Jakarta EE;
- JPA;
- Hibernate;
- Jackson;
- OpenAPI;
- SLF4J;
- APIs externas;
- Clientes HTTP;
- Frameworks de persistência.

## ✔ Correto

```java
public class SampleEntity { }
```

## ❌ Incorreto

```java
@Entity
public class SampleEntity { }
```

```java
public class SampleEntity {

    @JsonProperty
    private String value;
}
```

```java
@Slf4j
public class SampleEntity { }
```

---

# Resumo das Convenções

Toda implementação da camada Domain deve respeitar os seguintes princípios:

- O domínio não depende de nenhuma outra camada.
- Entidades devem herdar de `Entity`.
- Identificadores devem utilizar `IdValueObject`.
- Value Objects devem ser imutáveis.
- Value Objects devem ser criados, preferencialmente, através de `Result<T>`.
- Toda entidade deve validar seu estado através de `validateOrThrow()`.
- Regras de negócio pertencem exclusivamente ao Domain.
- Exceções devem herdar de `DomainException`.
- Não é permitido utilizar anotações ou dependências de frameworks.
- O domínio deve permanecer independente de detalhes técnicos.
