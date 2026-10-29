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
│   │   └── Entity.kt
│   ├── exception/
│   │   └── DomainException.kt
│   └── valueobject/
│       ├── IdValueObject.kt
│       └── exception/
│           └── InvalidIdValueObjectException.kt
└── sample/
    ├── SampleEntity.kt
    └── valueobject/
        ├── SampleNameValueObject.kt
        └── exception/
            └── InvalidSampleNameException.kt
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

Ela deve fornecer:

- Identificador (`IdValueObject`);
- Data de criação (`Instant`);
- Método `validateOrThrow()`;
- Igualdade baseada na identidade.

A classe base **não** modela `updatedAt`: atualização não é um comportamento universal — entidades imutáveis após a criação (eventos, registros de log, lançamentos) nunca são alteradas. Quando o agregado tiver ciclo de atualização, a própria entidade declara seu `updatedAt` (`Instant?`), como mostrado adiante.

## Estrutura de entidade de domínio

Entidades seguem este esquema:

1. **Campos declarados no corpo** (`val` para imutáveis, `var private set` para mutáveis).
2. **Construtor primário**: recebe todos os campos como primitivos/strings. O `createdAt` (`Instant?`) é opcional, com valor padrão `null`: quando a entidade é criada pela primeira vez vem `null` e a classe base assume `Instant.now()`; quando a entidade é reconstruída a partir da persistência, o mapper repassa o `createdAt` armazenado, preservando o valor original. O `createdAt` é, portanto, um fato do domínio — definido aqui, não gerado pela infraestrutura.
3. **`updatedAt` (quando o agregado tiver ciclo de atualização)**: declarado como `var updatedAt: Instant? = null; private set`. Começa `null` numa entidade recém-criada (nunca foi alterada) e passa a `Instant.now()` a cada método de negócio que muda o estado. O construtor primário recebe o `updatedAt` (`Instant?`, padrão `null`) para que a reconstrução a partir da persistência repasse o valor armazenado (que pode ser `null` ou um `Instant`). Assim como o `createdAt`, é um fato do domínio.
4. **Bloco `init`**: cria os value objects a partir dos parâmetros primitivos, chama `validateOrThrow` e atribui os campos do corpo (incluindo o `updatedAt` recebido, quando existir).
5. **Métodos de negócio** (`update*`, `markAs*`): recebem primitivos/strings, recriam o value object via `of(...).getValueResultOrThrow()`, atualizam o campo e registram a alteração com `this.updatedAt = Instant.now()`.

## ✔ Correto

```kotlin
class SampleEntity(
    id: String,
    name: String,
    createdAt: Instant? = null,
    updatedAt: Instant? = null
) : Entity(id, createdAt) {

    var name: SampleNameValueObject
        private set

    var updatedAt: Instant? = updatedAt
        private set

    init {
        val nameResult = SampleNameValueObject.of(name)

        val results = listOf(nameResult)
        validateOrThrow(results)

        this.name = nameResult.getValueResult()
    }

    fun updateName(name: String) {
        this.name = SampleNameValueObject.of(name).getValueResultOrThrow()
        this.updatedAt = Instant.now()
    }
}
```

## ✔ Classe base

```kotlin
abstract class Entity(id: String, createdAt: Instant? = null) {

    val id: IdValueObject = IdValueObject.of(id).getValueResultOrThrow()
    val createdAt: Instant = createdAt ?: Instant.now()

    protected fun validateOrThrow(results: List<Result<*>>) {
        val errors = results
            .filter { it.isFailure }
            .map { it.getErrorResult() as DomainException }

        if (errors.isNotEmpty()) {
            throw DomainException.with(errors)
        }
    }
}
```

As seguintes regras devem ser respeitadas:

- Toda entidade deve herdar de `Entity`.
- Entidades representam comportamento, não apenas dados.
- Não devem possuir setters públicos.
- Alterações de estado devem ocorrer através de métodos de negócio.
- Toda entidade deve validar seu estado durante sua construção via `validateOrThrow`.
- Nenhuma entidade pode permanecer inválida após sua criação.

## ✔ Correto

```kotlin
sample.updateName("New name")
```

## ❌ Incorreto

```kotlin
sample.name = SampleNameValueObject.of("New name")
```

## ❌ Incorreto

```kotlin
@Entity
@Table(name = "sample")
class SampleEntity { }
```

---

# Objetos Base

Objetos reutilizáveis entre agregados devem permanecer em `domain/common`.

Estrutura recomendada:

```text
common/
├── abstracts/
│   └── Entity.kt
├── exception/
│   └── DomainException.kt
└── valueobject/
    ├── IdValueObject.kt
    └── exception/
        └── InvalidIdValueObjectException.kt
```

As abstrações base recomendadas são:

- Entity
- DomainException
- IdValueObject

`IdValueObject` é um Value Object como qualquer outro e segue o mesmo arquétipo (construtor privado, factory `of(...)` retornando `Result`, `asString()`, `equals`/`hashCode` por valor). Sua validação é o formato do identificador — um UUID textual válido.

```kotlin
class IdValueObject private constructor(private val value: String) {

    fun asString(): String = value

    override fun equals(other: Any?): Boolean {
        if (this === other) return true

        if (other !is IdValueObject) return false

        return asString() == other.asString()
    }

    override fun hashCode(): Int = value.hashCode()

    companion object {
        fun of(value: String): Result<IdValueObject> {
            return runCatching { UUID.fromString(value) }
                .map { Result.success(IdValueObject(value)) }
                .getOrElse { Result.fail(InvalidIdValueObjectException()) }
        }
    }
}
```

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
- Construtor `private` — instanciação exclusiva via `of(...)` no companion object.
- Factory `of(value: String)` retorna `Result<VO>`.
- O valor primitivo é exposto por um método de conversão correspondente ao seu tipo (`asString()`, `asInt()`, `asLong()`, etc.). Nunca use `getValue()`.
- Sobrescrevem `equals` e `hashCode` com base no valor retornado pelo método de conversão.

## ✔ Correto

```kotlin
class SampleNameValueObject private constructor(private val value: String) {

    fun asString(): String = value

    override fun equals(other: Any?): Boolean {
        if (this === other) return true

        if (other !is SampleNameValueObject) return false

        return asString() == other.asString()
    }

    override fun hashCode(): Int = value.hashCode()

    companion object {
        private const val MAX_LENGTH = 255

        fun of(value: String): Result<SampleNameValueObject> {
            if (value.isBlank() || value.length > MAX_LENGTH) {
                return Result.fail(InvalidSampleNameException())
            }

            return Result.success(SampleNameValueObject(value))
        }
    }
}
```

## ❌ Incorreto

```kotlin
class SampleNameValueObject(var value: String) {
    fun setValue(value: String) {
        this.value = value
    }
}
```

---

# Result

A criação de **Value Objects** deve ocorrer através do objeto `Result`.

O `Result<T>` permite representar sucesso ou falha sem depender do lançamento imediato de exceções.

Ele se aplica à criação de Value Objects. **Entidades não usam `Result` na própria instanciação**: são criadas diretamente pelo construtor, que internamente cria os Value Objects (cada um via `of(...)`) e chama `validateOrThrow`. Portanto, "criar via `Result`" vale para os Value Objects, não para a entidade que os compõe.

> **Nota de API — nomes dos acessores.** O `Result` do projeto expõe `getValueResult()` (valor de sucesso), `getValueResultOrThrow()` (variante que lança) e `getErrorResult()` (erro de falha). Os nomes têm o sufixo `Result` de propósito: um `getValue()`/`getError()` colidiria com a convenção de getters do Kotlin (seriam interpretados como as propriedades `value`/`error`), causando conflito de compilação. Use sempre as variantes com sufixo `Result`; nunca `getValue()` ou `getError()`.

## ✔ Correto

```kotlin
val result = SampleNameValueObject.of(value)

if (result.isFailure) {
    throw result.getErrorResult()
}

val name = result.getValueResult()
```

## ✔ Correto

```kotlin
val name = SampleNameValueObject.of(value).getValueResultOrThrow()
```

## ❌ Incorreto

```kotlin
val name = SampleNameValueObject(value)  // construtor direto
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
- Devem ter uma mensagem customizada interna.
- Não devem representar detalhes técnicos.
- Não devem depender de infraestrutura.

Sobre a hierarquia: `DomainException` é, ela própria, filha de `RuntimeException` (exceção não checada). Toda exceção de domínio herda de `DomainException` e, por consequência, também é uma `RuntimeException` — essa é a base esperada. O que não é permitido é **lançar `RuntimeException` ou `IllegalArgumentException` cruas** (sem tipo de domínio) a partir do domínio: a violação de regra de negócio deve ser sempre expressa por uma subclasse descritiva de `DomainException`, nunca por uma exceção genérica.

## ✔ Correto

```kotlin
class InvalidSampleNameException : DomainException(
    "Sample name must not be blank and must have at most 255 characters"
)
```

## ❌ Incorreto — lançar exceção genérica em vez de uma exceção de domínio

```kotlin
if (value.isBlank()) {
    throw IllegalArgumentException("invalid sample name")
}
```

## ❌ Incorreto — exceção de domínio representando detalhe técnico

```kotlin
class SQLException : RuntimeException()
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

```kotlin
class SampleValidationService {

    fun validate(sample: SampleEntity) {
        ...
    }
}
```

## ❌ Incorreto

```kotlin
class SampleService(
    private val repository: SampleRepository
)
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

```kotlin
class SampleEntity(
    id: String,
    name: String
) : Entity(id)
```

## ❌ Incorreto

```kotlin
@Entity
class SampleEntity { }
```

```kotlin
@Slf4j
class SampleEntity { }
```

---

# Resumo das Convenções

Toda implementação da camada Domain deve respeitar os seguintes princípios:

- O domínio não depende de nenhuma outra camada.
- Entidades devem herdar de `Entity`.
- Identificadores devem utilizar `IdValueObject`.
- Value Objects devem ser imutáveis.
- Value Objects devem ser criados através de `Result<T>` via companion object `of(...)`; entidades são criadas diretamente pelo construtor, que cria os Value Objects e valida via `validateOrThrow`.
- Toda entidade deve validar seu estado através de `validateOrThrow()`.
- Regras de negócio pertencem exclusivamente ao Domain.
- Exceções devem herdar de `DomainException`.
- Não é permitido utilizar anotações ou dependências de frameworks.
- O domínio deve permanecer independente de detalhes técnicos.
