# Convenções da Camada Application

A camada **Application** é responsável por orquestrar os casos de uso da aplicação.

Ela coordena a execução das regras de negócio da camada Domain e realiza validações ou decisões que dependam de recursos externos.

Regras que possam ser implementadas exclusivamente pelo domínio não devem ser implementadas nesta camada.

---

# Estrutura

A organização da camada Application deve refletir os casos de uso da aplicação.

A estrutura recomendada é:

```text
application/
├── port/
│   ├── persistence/
│   │   └── repository/
│   └── security/
│       ├── dto/
│       └── exception/
└── usecase/
    └── sample/
        ├── dto/
        │   ├── input/
        │   └── output/
        ├── exception/
        └── CreateSampleUseCase.kt
```

As seguintes regras devem ser respeitadas:

- Cada contexto deve possuir seu próprio pacote.
- Cada caso de uso é uma **classe concreta**, sem interface e sem sufixo `Impl`, diretamente no pacote do contexto (`usecase/<contexto>`).
- Não devem existir subpastas `interfaces/` ou `implementation/`.
- DTOs devem permanecer separados entre entrada e saída.
- Portas devem permanecer organizadas por responsabilidade.

---

# Casos de Uso

Casos de uso representam operações executadas pela aplicação.

Cada caso de uso deve representar apenas uma responsabilidade.

Um caso de uso tem exatamente uma implementação e nenhum ponto de variação real. A interface single-impl é cerimônia que não compra desacoplamento, portanto o caso de uso é modelado como uma **classe concreta**. Interfaces (contratos) são reservadas às **portas** — repositórios, segurança e gateways —, onde existe inversão de dependência real com a infraestrutura.

As seguintes regras devem ser respeitadas:

- Todo caso de uso é uma classe concreta, sem interface e sem sufixo `Impl`.
- O método público não usa `override` — não há contrato para sobrescrever.
- Injeção pelo construtor primário, com dependências imutáveis (`private val`).
- Cada caso de uso deve executar apenas um fluxo da aplicação.
- Casos de uso não devem conter regras de infraestrutura.
- Casos de uso não devem depender da camada Presentation.

## ✔ Correto

```kotlin
@Service
class CreateSampleUseCase(
    private val sampleRepository: SampleRepository
) {
    fun execute(input: CreateSampleInputDto): CreateSampleOutputDto {
        ...
    }
}
```

## ❌ Incorreto — interface single-impl e sufixo Impl

```kotlin
interface CreateSampleUseCase {
    fun execute(input: CreateSampleInputDto): CreateSampleOutputDto
}

@Service
class CreateSampleUseCaseImpl(
    private val sampleRepository: SampleRepository
) : CreateSampleUseCase {
    ...
}
```

## ❌ Incorreto — serviço com múltiplas responsabilidades

```kotlin
@Service
class SampleService {
    fun create(...) {}
    fun update(...) {}
    fun delete(...) {}
}
```

---

# DTOs

DTOs representam exclusivamente os dados trafegados entre camadas.

Não representam regras de negócio.

As seguintes regras devem ser respeitadas:

- Devem ser imutáveis.
- Devem utilizar `data class`.
- Não devem possuir validações.
- Não devem possuir regras de negócio.
- Não devem possuir dependência de frameworks.
- Devem receber apenas tipos primitivos ou strings — nunca value objects.

## ✔ Correto

```kotlin
data class CreateSampleInputDto(
    val name: String
)
```

```kotlin
data class CreateSampleOutputDto(
    val id: String,
    val name: String
)
```

## ❌ Incorreto

```kotlin
data class CreateSampleInputDto(
    @field:NotBlank
    val name: String
)
```

---

# Portas

Portas representam contratos entre a camada Application e componentes externos.

Toda comunicação com infraestrutura deve ocorrer através de portas.

As seguintes regras devem ser respeitadas:

- Portas devem ser interfaces.
- Portas devem representar comportamento.
- Não devem possuir implementação.
- Devem permanecer organizadas por responsabilidade.
- Assinaturas de repositórios trabalham exclusivamente com tipos do domínio: `Entity`, `ValueObject`. Nunca expõem tipos JPA.

## ✔ Correto

```kotlin
interface SampleRepository {
    fun save(sample: SampleEntity): SampleEntity
    fun findById(id: IdValueObject): SampleEntity?
    fun existsByName(name: SampleNameValueObject): Boolean
}
```

```kotlin
interface PasswordEncoder {
    fun encode(value: String): String
    fun matches(raw: String, encoded: String): Boolean
}
```

## ❌ Incorreto

```kotlin
class SampleRepository { }
```

---

## ❌ Incorreto

```kotlin
@Service
class SampleService {
    fun create() {}
    fun update() {}
    fun delete() {}
}
```

---

# Fluxo de Dados

A camada Application é responsável por coordenar o fluxo entre as portas e o domínio.

O fluxo recomendado é:

```text
Input DTO
      ↓
Caso de Uso
      ↓
Value Objects
      ↓
Entidade
      ↓
Porta
      ↓
Output DTO
```

As seguintes regras devem ser respeitadas:

- O caso de uso recebe um DTO de entrada.
- O caso de uso instancia a entidade diretamente via construtor público.
- O caso de uso utiliza apenas portas para acessar infraestrutura.
- O retorno deve ocorrer através de um DTO de saída.
- A conversão da entidade para Output DTO deve ser feita por um método privado `toOutput(entity)`.

## ✔ Correto

```kotlin
fun execute(input: CreateSampleInputDto): CreateSampleOutputDto {
    val sample = SampleEntity(
        id = UUID.randomUUID().toString(),
        name = input.name
    )

    val saved = sampleRepository.save(sample)

    return toOutput(saved)
}

private fun toOutput(sample: SampleEntity): CreateSampleOutputDto =
    CreateSampleOutputDto(
        id = sample.id.asString(),
        name = sample.name.asString()
    )
```

---

# Validação de Value Objects

Sempre que o input do use case contiver um campo que será usado **isoladamente** — como um ID para busca ou um campo para verificação de unicidade —, use `getValueResultOrThrow()` diretamente.

Quando todos os campos necessários estão disponíveis e a entidade será criada via construtor **sem verificação de negócio prévia**, não valide os value objects manualmente. O construtor já chama `validateOrThrow` internamente.

## ✔ Correto — busca por ID

```kotlin
val sampleIdValue = IdValueObject.of(input.sampleId).getValueResultOrThrow()
val sample = sampleRepository.findById(sampleIdValue) ?: throw SampleNotFoundException()
```

## ✔ Correto — verificação de unicidade antes da criação

```kotlin
val nameResult = SampleNameValueObject.of(input.name).getValueResultOrThrow()
if (sampleRepository.existsByName(nameResult)) {
    throw SampleNameAlreadyExistsException()
}
val sample = SampleEntity(id = UUID.randomUUID().toString(), name = input.name)
```

## ❌ Incorreto — validação duplicada desnecessária

```kotlin
val nameResult = SampleNameValueObject.of(input.name).getValueResultOrThrow()
val sample = SampleEntity(id = UUID.randomUUID().toString(), name = input.name)
```

---

# Conversões

A camada Application é responsável pela conversão entre DTOs e objetos do domínio.

As seguintes regras devem ser respeitadas:

- DTOs nunca devem atravessar para a camada Domain.
- Entidades nunca devem ser utilizadas diretamente pela camada Presentation.
- A conversão de entidade para Output DTO deve ser centralizada em um método privado `toOutput` dentro do caso de uso.
- Nunca repita o mapeamento inline nem exponha entidades de domínio fora do caso de uso.

## ✔ Correto

```kotlin
private fun toOutput(sample: SampleEntity): CreateSampleOutputDto =
    CreateSampleOutputDto(
        id = sample.id.asString(),
        name = sample.name.asString()
    )
```

## ❌ Incorreto

```kotlin
return sample
```

---

# Exceções de Caso de Uso

Exceções específicas de cada domínio de caso de uso ficam em `exception/`, ao lado das classes de caso de uso e da pasta `dto/`.

As seguintes regras devem ser respeitadas:

- Estendem `RuntimeException` diretamente (não `DomainException`).
- Carregam uma mensagem fixa.
- Devem possuir nomes descritivos.

## ✔ Correto

```kotlin
class SampleNotFoundException : RuntimeException("Sample not found")
class SampleAccessDeniedException : RuntimeException("Access denied to this sample")
class SampleNameAlreadyExistsException : RuntimeException("Sample name already exists")
```

---

# Gerenciamento Transacional

A fronteira transacional de um caso de uso é uma responsabilidade de **orquestração** — decidir se o conjunto de operações contra as portas de persistência deve ser confirmado por inteiro ou revertido por inteiro. Por ser orquestração, essa decisão pertence ao próprio caso de uso.

Por esse motivo abrimos uma **exceção arquitetural consciente**: `org.springframework.transaction.annotation.Transactional` é a **única** dependência de framework de persistência permitida na camada Application. A anotação é aplicada **diretamente no método `execute`** do caso de uso.

A alternativa — abstrair a transação por uma porta dedicada (ex.: `TransactionExecutor`/`UnitOfWork`) — adicionaria indireção sem ganho prático, já que a fronteira transacional não varia por tecnologia dentro deste projeto. Essa exceção está registrada também no documento de arquitetura (`convention-architecture.md`), para deixar explícito que é uma decisão deliberada, e não um descuido.

## Critério

O gatilho da regra é o **número de invocações de métodos de porta de persistência** dentro do `execute` — **não** o número de comandos SQL gerados. Uma única invocação de porta pode gerar vários SQLs e ainda assim conta como **uma só** invocação.

A razão de o número de invocações ser o gatilho: com **uma** invocação não há nada a coordenar; a partir de **duas** surge o risco de o conjunto falhar no meio (a primeira confirma, a segunda falha, banco inconsistente). O `@Transactional` transforma as N operações num bloco indivisível, confirmado por inteiro no commit único ao fim do método ou revertido por inteiro no rollback.

## Regras

- **1 invocação de porta de persistência → SEM `@Transactional`.** A operação isolada já é atômica pela transação do próprio Spring Data. Anotar não é erro, mas não agrega.
- **2 ou mais invocações, TODAS de leitura → `@Transactional(readOnly = true)`.** Garante um snapshot consistente entre as leituras e sinaliza a intenção de somente leitura.
- **2 ou mais invocações, contendo AO MENOS UMA escrita → `@Transactional`.** Caso crítico: sem a transação abrangente, uma falha após a primeira escrita deixaria o banco em estado parcial e irreversível. O critério "ao menos uma escrita" cobre tanto leitura + escrita quanto escrita + escrita.

## ✔ Correto — 1 invocação (sem anotação)

```kotlin
@Service
class ListTasksUseCase(
    private val taskRepository: TaskRepository
) {
    fun execute(input: ListTasksInputDto): List<TaskOutputDto> {
        val userIdResult = IdValueObject.of(input.userId).getValueResultOrThrow()

        return taskRepository.findAllByUserId(userIdResult).map(::toOutput)
    }

    private fun toOutput(task: TaskEntity): TaskOutputDto = ...
}
```

## ✔ Correto — 2+ invocações, todas de leitura (`readOnly = true`)

```kotlin
@Service
class GetTaskSummaryUseCase(
    private val taskRepository: TaskRepository,
    private val userRepository: UserRepository
) {
    @Transactional(readOnly = true)
    fun execute(input: GetTaskSummaryInputDto): TaskSummaryOutputDto {
        val userIdResult = IdValueObject.of(input.userId).getValueResultOrThrow()

        val user = userRepository.findById(userIdResult) ?: throw UserNotFoundException()
        val tasks = taskRepository.findAllByUserId(userIdResult)

        return toOutput(user, tasks)
    }

    private fun toOutput(user: UserEntity, tasks: List<TaskEntity>): TaskSummaryOutputDto = ...
}
```

## ✔ Correto — 2+ invocações com ao menos uma escrita (`@Transactional`)

```kotlin
@Service
class CreateUserUseCase(
    private val userRepository: UserRepository,
    private val passwordEncoder: PasswordEncoder
) {
    @Transactional
    fun execute(input: CreateUserInputDto): CreateUserOutputDto {
        val encodedPassword = passwordEncoder.encode(input.password)
        val user = UserEntity(
            id = UUID.randomUUID().toString(),
            username = input.username,
            password = encodedPassword
        )

        if (userRepository.existsByUsername(user.username)) {
            throw UsernameAlreadyExistsException()
        }

        val saved = userRepository.save(user)

        return toOutput(saved)
    }

    private fun toOutput(user: UserEntity): CreateUserOutputDto = ...
}
```

> Note que `passwordEncoder.encode(...)` é uma porta de **segurança**, não de persistência; ela não entra na contagem. As invocações de persistência aqui são `existsByUsername` (leitura) e `save` (escrita) — duas invocações com ao menos uma escrita, logo `@Transactional`.

## ❌ Incorreto — 2+ escritas sem transação abrangente

```kotlin
@Service
class UpdateTaskUseCase(
    private val taskRepository: TaskRepository
) {
    // Faltando @Transactional: se o segundo save falhar,
    // o primeiro já foi confirmado e o banco fica inconsistente.
    fun execute(input: UpdateTaskInputDto): TaskOutputDto {
        val task = taskRepository.save(...)
        val audit = taskRepository.save(...)
        return toOutput(task)
    }
}
```

## ❌ Incorreto — `@Transactional` em caso de uso com 1 única invocação

```kotlin
@Service
class ListTasksUseCase(
    private val taskRepository: TaskRepository
) {
    @Transactional(readOnly = true) // Desnecessário: há apenas uma invocação de porta.
    fun execute(input: ListTasksInputDto): List<TaskOutputDto> {
        val userIdResult = IdValueObject.of(input.userId).getValueResultOrThrow()
        return taskRepository.findAllByUserId(userIdResult).map(::toOutput)
    }
}
```

---

# Dependências

A camada Application pode depender apenas da camada Domain e de contratos definidos pela própria Application.

Não é permitido depender de:

- Controllers;
- Requests;
- Responses;
- Models de persistência;
- Repositórios concretos;
- Frameworks de persistência — **com uma única exceção consciente: a anotação `org.springframework.transaction.annotation.Transactional`, usada no método `execute` do caso de uso conforme a seção [Gerenciamento Transacional](#gerenciamento-transacional)**;
- Componentes da camada External.

---

# Resumo das Convenções

Toda implementação da camada Application deve respeitar os seguintes princípios:

- Cada caso de uso representa uma única operação da aplicação.
- Todo caso de uso é uma classe concreta, sem interface e sem sufixo `Impl`.
- Interface existe para inverter dependência de infraestrutura (portas), não para envelopar componente de implementação única.
- DTOs devem utilizar `data class`.
- DTOs não possuem regras de negócio.
- Toda comunicação externa deve ocorrer através de portas.
- A camada Application coordena o fluxo da aplicação, executa validações que dependam de recursos externos e orquestra a execução das regras de negócio da camada Domain.
- Entidades pertencem ao Domain.
- Componentes da External nunca devem ser acessados diretamente.
- O retorno dos casos de uso deve ocorrer através de DTOs de saída.
- A conversão para Output DTO deve ser centralizada em um método privado `toOutput`.
- A fronteira transacional é definida no `execute` do caso de uso: `@Transactional` é a única dependência de framework de persistência permitida, aplicada conforme o número de invocações de portas de persistência — 1 invocação sem anotação; 2+ apenas de leitura com `@Transactional(readOnly = true)`; 2+ com ao menos uma escrita com `@Transactional` (ver seção Gerenciamento Transacional).
