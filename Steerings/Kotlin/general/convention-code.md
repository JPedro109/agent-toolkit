# Convenções de Código

# Nomenclatura

A nomenclatura dos componentes deve ser consistente, descritiva e refletir claramente sua responsabilidade.

As seguintes convenções devem ser adotadas:

- Classes devem utilizar **PascalCase**.
- Métodos e propriedades devem utilizar **camelCase**.
- Constantes devem utilizar **UPPER_SNAKE_CASE**.
- Pacotes devem utilizar apenas letras minúsculas.
- Interfaces devem representar comportamentos ou contratos.
- Interfaces **não devem** utilizar o prefixo `I`.
- Uma classe concreta só recebe o sufixo `Impl` quando implementa uma interface homônima que representa um contrato real com mais de um implementador possível — o caso das portas e seus adaptadores.
- Casos de uso não possuem interface e, portanto, não utilizam o sufixo `Impl`. São nomeados pela operação que representam, terminando em `UseCase`.
- Adaptadores de infraestrutura (implementações de portas) utilizam o sufixo `Adapter` (ou um nome que descreva a tecnologia, ex: `JpaSampleRepository`).
- Siglas e acrônimos em identificadores seguem o estilo idiomático do Kotlin: apenas a primeira letra em maiúscula, tratando a sigla como uma palavra comum. Use `Dto`, `Id`, `Http`, `Json`, `Url` — nunca `DTO`, `ID`, `HTTP`. Ex.: `CreateSampleInputDto`, `SampleOutputDto`, `DecodeTokenDto`.
- Classes concretas devem possuir nomes que representem claramente sua responsabilidade.

Evite abreviações desnecessárias e nomes genéricos que não expressem claramente a responsabilidade do componente.

### ✔ Correto

```kotlin
// Caso de uso: classe concreta, sem interface, nomeada pela operação + sufixo UseCase.
class CreateSampleUseCase

// Porta: interface que representa o contrato.
interface PasswordEncoder

// Adaptador com nome que descreve a tecnologia que implementa a porta.
class BCryptPasswordEncoder : PasswordEncoder

// Adaptador com sufixo Adapter quando o nome da tecnologia não agrega.
class SampleRepositoryAdapter : SampleRepository
```

### ❌ Incorreto — interface single-impl e sufixo `Impl` em caso de uso

```kotlin
interface CreateSampleUseCase

class CreateSampleUseCaseImpl : CreateSampleUseCase
```

### ❌ Incorreto — nomes genéricos

```kotlin
class SampleManager

class Helper

class Util

class Processor
```

---

# Constantes

Valores reutilizados ou que representem regras estáticas devem ser declarados como constantes.

Não é permitido utilizar valores literais ("magic numbers" ou "magic strings") quando seu significado puder ser representado por uma constante nomeada.

Constantes devem ser declaradas no `companion object` com `const val` para primitivos ou `private val` para tipos complexos.

A constante deve morar no componente dono da regra que ela representa. Um limite que faz parte da invariante de um Value Object (por exemplo, o comprimento máximo de um nome) pertence ao próprio Value Object e **não** deve ser duplicado em entidades, casos de uso ou configurações. Ver `convention-layer-domain.md`, onde `SampleNameValueObject` declara seu próprio `MAX_LENGTH`.

### ✔ Correto

```kotlin
companion object {
    private const val MAX_LOGIN_ATTEMPTS = 3
}
```

### ❌ Incorreto

```kotlin
if (attempts > 3) {
    ...
}
```

---

# Imutabilidade e uso de `val`

Sempre prefira `val` para declarar variáveis. Use `var` apenas em casos extremos onde a reatribuição seja estritamente necessária e não haja como modelar o problema com imutabilidade.

### ✔ Correto

```kotlin
val sample = sampleRepository.findById(sampleIdValue) ?: throw SampleNotFoundException()
val output = toOutput(sample)
```

### ❌ Incorreto

```kotlin
var sample = sampleRepository.findById(sampleIdValue) ?: throw SampleNotFoundException()
var output = toOutput(sample)
```

---

# Named Parameters

Sempre use named parameters ao instanciar classes, chamar construtores ou invocar funções com mais de um argumento. Isso melhora a legibilidade e evita erros de posicionamento.

### ✔ Correto

```kotlin
val input = CreateSampleInputDto(name = name.asString())
val sample = SampleEntity(id = UUID.randomUUID().toString(), name = input.name)
```

### ❌ Incorreto

```kotlin
val input = CreateSampleInputDto(name.asString())
val sample = SampleEntity(UUID.randomUUID().toString(), input.name)
```

Não há exceções a esta regra.

---

# Formatação

Todo o código deve seguir um único padrão de formatação.

A formatação deve garantir:

- Consistência visual;
- Facilidade de leitura;
- Padronização entre todos os projetos.

A formatação deve ser automatizada por **ktlint**.

Além da formatação automática, o código deve ser organizado em **blocos lógicos**, utilizando linhas em branco para separar etapas distintas de uma operação.

Cada bloco deve representar uma fase claramente identificável da execução, como:

- Validação;
- Conversão de dados;
- Execução da regra de negócio;
- Persistência;
- Publicação de eventos;
- Retorno do resultado.

### ✔ Correto

```kotlin
fun execute(input: CreateSampleInputDto): CreateSampleOutputDto {
    val nameResult = SampleNameValueObject.of(input.name).getValueResultOrThrow()

    if (sampleRepository.existsByName(nameResult)) {
        throw SampleNameAlreadyExistsException()
    }

    val sample = SampleEntity(
        id = UUID.randomUUID().toString(),
        name = input.name
    )

    val saved = sampleRepository.save(sample)

    return toOutput(saved)
}
```

### ❌ Incorreto

```kotlin
fun execute(input: CreateSampleInputDto): CreateSampleOutputDto {
    val nameResult = SampleNameValueObject.of(input.name).getValueResultOrThrow()
    if (sampleRepository.existsByName(nameResult)) {
        throw SampleNameAlreadyExistsException()
    }
    val sample = SampleEntity(id = UUID.randomUUID().toString(), name = input.name)
    val saved = sampleRepository.save(sample)
    return toOutput(saved)
}
```

---

# Ordenação dos Membros

Os membros de uma classe devem seguir uma ordem lógica e consistente.

A ordem recomendada é:

1. Campos de instância / propriedades;
2. Construtores / bloco `init`;
3. Métodos públicos;
4. Métodos protegidos;
5. Métodos privados;
6. `companion object` (constantes).

Métodos auxiliares devem permanecer próximos dos métodos que os utilizam, mas sempre métodos privados devem ficar abaixo dos públicos.

> **Value Objects e entidades do domínio seguem o arquétipo descrito em `convention-layer-domain.md`**, não esta ordenação genérica. No Value Object, o `companion object` com a factory `of(...)` é a API de entrada e coexiste com `equals`/`hashCode`; na entidade, o bloco `init` vem logo após os campos. Esta seção aplica-se a serviços, casos de uso, controllers e adaptadores.

### ✔ Correto

```kotlin
@Service
class CreateSampleUseCase(
    private val sampleRepository: SampleRepository
) {

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
}
```

---

# Organização dos Métodos

Cada método deve possuir apenas uma responsabilidade.

Sempre que possível:

- Métodos públicos devem representar operações de alto nível.
- Detalhes de implementação devem ser extraídos para métodos privados.
- Métodos devem ser pequenos e objetivos.
- Um método não deve executar múltiplas responsabilidades.

---

# Injeção de Dependência

Toda dependência entre componentes deve ser realizada por meio de injeção de dependência via construtor primário.

As seguintes regras devem ser respeitadas:

- A injeção deve ocorrer exclusivamente pelo construtor primário.
- Dependências obrigatórias devem ser imutáveis (`private val`).
- Instanciações diretas devem ser evitadas para componentes gerenciados pela aplicação.
- Dependências devem ser representadas por contratos sempre que possível.
- Nunca utilizar `@Autowired` em campo ou setter.

### ✔ Correto

```kotlin
@Service
class CreateSampleUseCase(
    private val sampleRepository: SampleRepository,
    private val passwordEncoder: PasswordEncoder
) {
    ...
}
```

### ❌ Incorreto

```kotlin
@Service
class CreateSampleUseCase {
    @Autowired
    private lateinit var sampleRepository: SampleRepository

    @Autowired
    private lateinit var passwordEncoder: PasswordEncoder
}
```

---

# Logging

Logs devem registrar informações relevantes para auditoria, monitoramento e diagnóstico da aplicação.

As seguintes práticas devem ser adotadas:

- Registrar apenas eventos relevantes para o funcionamento da aplicação.
- Utilizar o nível de log apropriado (`TRACE`, `DEBUG`, `INFO`, `WARN` ou `ERROR`).
- Utilizar mensagens objetivas e descritivas.
- Nunca registrar dados sensíveis, como senhas, tokens, chaves ou informações pessoais.
- Nunca utilizar logs como mecanismo de controle de fluxo.
- Nunca registrar identificadores de rastreamento (Trace ID, Span ID, Request ID, etc.), pois essas informações já são propagadas automaticamente pelo Baggage Field.
- Nunca registrar a mesma exceção em múltiplas camadas da aplicação.
- Utilizar string interpolation do Kotlin (`"$variavel"`) nos logs — não usar placeholders `{}` do SLF4J.
- Registrar exceções apenas no ponto em que elas forem efetivamente tratadas ou onde houver contexto relevante a ser acrescentado.
- Logs de operações devem ser realizados na camada de Presentation, registrando as entradas (requests) e saídas (responses) quando apropriado.
- Logs de exceções devem ser realizados exclusivamente pelo Global Exception Handler.
- A única exceção ocorre quando uma exceção de infraestrutura precisa ser traduzida para uma exceção da aplicação ou do domínio. Nesse caso, a exceção original deve ser registrada antes da tradução.
- Utilizar exclusivamente o Logger do SLF4J para registro de logs.
- Não utilizar `println`, `System.out`, `System.err` ou `printStackTrace()`.

### ✔ Correto

```kotlin
logger.info("Creating sample - request: $request")
```

```kotlin
logger.info("Creating sample - response: $response")
```

```kotlin
logger.error("Resource not found: ${ex.message}", ex)
```

### ❌ Incorreto

```kotlin
logger.info("User $username authenticated using password $password")
```

```kotlin
logger.info("Sample created: " + sampleName)
```

```kotlin
logger.info("RequestId=$requestId TraceId=$traceId Sample created.")
```

- A instância do logger sempre deve ser criada no `companion object`.

### Exemplo

```kotlin
@RestController
@RequestMapping("/api/v1/samples")
class SampleController(
    private val createSampleUseCase: CreateSampleUseCase
) : SampleControllerDoc {

    companion object {
        private val logger = LoggerFactory.getLogger(SampleController::class.java)
    }
}
```

---

# Configurações

As configurações da aplicação devem permanecer centralizadas e desacopladas da lógica de negócio.

As seguintes regras devem ser respeitadas:

- Configurações devem ser externalizadas.
- Valores configuráveis não devem ser codificados diretamente na aplicação.
- Configurações devem ser agrupadas conforme sua responsabilidade.
- Configurações tipadas devem utilizar `@ConfigurationProperties`.
- Classes de configuração devem representar apenas um grupo específico de propriedades.
- Regras de negócio não devem depender diretamente da origem das configurações.
- Componentes devem acessar apenas as configurações necessárias para sua responsabilidade.
- Classes de configuração sempre devem ter o sufixo `ConfigProperties`.
- Quando um prefixo contém subgrupos aninhados no YAML, use uma `data class` interna para o subgrupo.
- Todas as classes `@ConfigurationProperties` devem ser registradas centralmente com `@EnableConfigurationProperties` na classe principal da aplicação.

### ✔ Correto

```kotlin
@ConfigurationProperties(prefix = "security")
data class SecurityConfigProperties(
    val jwt: Jwt
) {
    data class Jwt(
        val secret: String,
        val accessTokenExpirationMs: Long,
        val refreshTokenExpirationMs: Long
    )
}
```

```kotlin
@Service
class TokenAdapter(
    private val properties: SecurityConfigProperties
) : Token {
    ...
}
```

### ❌ Incorreto

```kotlin
class TokenAdapter {
    companion object {
        private const val ISSUER = "My Application"
        private const val EXPIRATION = 86400000L
    }
}
```

### ❌ Incorreto

```kotlin
@Service
class TokenAdapter {
    @Value("\${security.jwt.secret}")
    private lateinit var secret: String

    @Value("\${security.jwt.access-token-expiration-ms}")
    private var expiration: Long = 0
}
```

---

# Imports

As instruções de importação devem permanecer organizadas e conter apenas dependências utilizadas pela classe.

As seguintes regras devem ser respeitadas:

- Imports não utilizados devem ser removidos.
- Imports curinga (`*`) não são permitidos.
- A organização dos imports deve seguir o padrão definido pelo ktlint.

### ✔ Correto

```kotlin
import java.time.Instant
import java.util.UUID
import org.springframework.stereotype.Service
```

### ❌ Incorreto

```kotlin
import java.util.*
```

---

# Análise Estática

Todo o código deve ser validado por ferramentas de análise estática antes de sua integração ao projeto.

As seguintes regras devem ser respeitadas:

- O projeto deve possuir configurações padronizadas de **ktlint** (formatação) e **detekt** (análise estática).
- Todo o código deve estar em conformidade com as regras definidas por essas configurações.
- Violações não devem ser ignoradas ou desabilitadas sem justificativa documentada.
- Política de zero warnings em ambas as ferramentas.
- Alterações no conjunto de regras devem ser revisadas e aprovadas pela equipe.

A configuração das ferramentas de análise estática faz parte da arquitetura do projeto e deve ser compartilhada entre todos os desenvolvedores.
