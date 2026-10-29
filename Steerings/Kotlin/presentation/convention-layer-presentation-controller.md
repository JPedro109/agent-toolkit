# Convenções da Camada Presentation — Controller (HTTP)

Este documento detalha o adaptador de entrada **HTTP** (controllers) da camada Presentation. Ele é uma extensão do documento base `convention-layer-presentation.md` e assume todos os princípios transversais definidos lá (sem regra de negócio, dependência exclusiva da camada Application, orquestração em quatro passos, tratamento de erro na borda e observabilidade).

O acesso HTTP recebe requisições, converte os dados para os DTOs da camada Application, executa os casos de uso e transforma o resultado em uma resposta HTTP apropriada.

---

# Estrutura

A organização do acesso HTTP deve refletir os recursos expostos pela aplicação.

A estrutura recomendada é:

```text
presentation/
└── controller/
    ├── SampleController.kt
    ├── documentation/
    │   └── payload/
    │       └── sample/
    │           ├── request/
    │           └── response/
    ├── payload/
    │   └── sample/
    │       ├── request/
    │       └── response/
    └── common/
        ├── handler/
        └── resolver/
```

> Filtros de requisição (ex.: autenticação JWT) **não** residem aqui. Por serem configuração técnica de bootstrap do framework, vivem em `configuration/filter/`, fora de `core/`, conforme o documento de arquitetura.

As seguintes regras devem ser respeitadas:

- Cada recurso deve possuir seu próprio controller.
- Payloads devem permanecer separados entre Request e Response.
- Componentes compartilhados devem permanecer em `common`.
- Documentação OpenAPI deve permanecer em `documentation`.

---

# Controllers

Controllers representam o ponto de entrada da aplicação.

Seu único objetivo é receber requisições, delegar a execução para um caso de uso e construir a resposta HTTP.

As seguintes regras devem ser respeitadas:

- Controllers não devem implementar regras de negócio.
- Controllers não devem acessar repositórios.
- Controllers não devem acessar adaptadores de infraestrutura.
- Controllers devem depender exclusivamente de casos de uso (classes concretas da camada Application).
- Controllers devem permanecer pequenos e objetivos.
- Controllers devem implementar a interface `*ControllerDoc` correspondente (nenhuma anotação do SpringDoc no corpo do controller).
- A interface `*ControllerDoc` **não** declara `@RequestMapping` — o path fica exclusivamente no controller.
- Logger declarado no `companion object` com `LoggerFactory.getLogger(XxxController::class.java)`.
- Controllers devem ter o log do início da requisição com a request se existir.
- Controllers devem ter o log do fim da requisição com a response se existir.
- Usar string interpolation do Kotlin (`"$variavel"`) nos logs.
- `AuthenticatedUserResolver.getUserId()` é o único ponto de extração do ID do usuário autenticado. Nunca acesse o `SecurityContext` diretamente nos controllers.

## ✔ Correto

```kotlin
@RestController
@RequestMapping("/api/v1/samples")
class SampleController(
    private val createSampleUseCase: CreateSampleUseCase
) : SampleControllerDoc {

    override fun createSample(@Valid @RequestBody request: CreateSampleRequest): ResponseEntity<SampleResponse> {
        logger.info("Creating sample - request: $request")

        val input = request.toInputDto()

        val output = createSampleUseCase.execute(input)

        val response = SampleResponse.of(output)

        logger.info("Creating sample - response: $response")
        return ResponseEntity.status(HttpStatus.CREATED).body(response)
    }

    companion object {
        private val logger = LoggerFactory.getLogger(SampleController::class.java)
    }
}
```

## ❌ Incorreto

```kotlin
@RestController
class SampleController {

    @Autowired
    private lateinit var sampleRepository: SampleRepository
}
```

## ❌ Incorreto

```kotlin
@PostMapping
fun create(@RequestBody request: CreateSampleRequest): ResponseEntity<*> {
    if (repository.existsByName(request.name)) {
        throw SampleAlreadyExistsException()
    }
}
```

---

# Payloads

Payloads representam exclusivamente os dados trafegados pela API.

Eles definem o contrato HTTP da aplicação e não representam entidades ou regras de negócio.

## Payloads de Request

Payloads de Request representam os dados recebidos pela API.

As seguintes regras devem ser respeitadas:

- Devem utilizar `data class`.
- Devem representar exclusivamente o contrato HTTP.
- Devem ser imutáveis.
- Não devem possuir regras de negócio.
- Não devem possuir dependência da camada Domain.
- Devem utilizar Bean Validation para validações sintáticas e estruturais (`@field:NotBlank`, `@field:Size`, etc.).
- Não devem conter lógica de conversão para objetos do domínio.
- Toda classe de Request deve possuir um método `toInputDto` responsável por converter o payload HTTP para o Input DTO do caso de uso. A conversão é apenas cópia de campos — sem validação de negócio, sem transformação com regra (o espelho do `of` do Response, no sentido da entrada).
- Controllers não devem montar manualmente o Input DTO a partir do Request.
- Devem implementar a interface `*RequestDoc` correspondente.
- Devem sobrescrever `toString()` quando contiverem dados sensíveis (senhas, tokens).

### ✔ Correto

```kotlin
data class CreateSampleRequest(
    @field:NotBlank
    @field:Size(max = 255)
    override val name: String
) : CreateSampleRequestDoc {

    fun toInputDto(): CreateSampleInputDto =
        CreateSampleInputDto(name = name)
}
```

### ✔ Correto — com dados sensíveis

```kotlin
data class SampleLoginRequest(
    @field:NotBlank
    @field:Size(min = 3, max = 50)
    override val username: String,
    @field:NotBlank
    @field:Size(min = 8, max = 128)
    override val password: String
) : SampleLoginRequestDoc {

    override fun toString(): String =
        "SampleLoginRequest{username='$username', password='[PROTECTED]'}"
}
```

### ❌ Incorreto

```kotlin
data class CreateSampleRequest(val name: String) {

    fun validate() {
        if (name.startsWith("ADMIN")) {
            throw DomainException()
        }
    }
}
```

---

## Payloads de Response

Payloads de Response representam exclusivamente os dados retornados pela API.

As seguintes regras devem ser respeitadas:

- Devem utilizar `data class`.
- Devem representar exclusivamente o contrato HTTP.
- Devem ser imutáveis.
- Não devem possuir regras de negócio.
- Não devem possuir dependência da camada Domain.
- Devem implementar a interface `*ResponseDoc` correspondente.
- Toda classe de Response deve possuir um método estático `of` no `companion object` responsável por converter o DTO de saída do caso de uso para o payload HTTP.
- Controllers não devem realizar manualmente o mapeamento entre Output DTO e Response.
- Devem sobrescrever `toString()` quando contiverem dados sensíveis.

### ✔ Correto

```kotlin
data class SampleResponse(
    override val id: String,
    override val name: String
) : SampleResponseDoc {

    companion object {
        fun of(output: SampleOutputDto): SampleResponse =
            SampleResponse(
                id = output.id,
                name = output.name
            )
    }
}
```

### ❌ Incorreto

```kotlin
val response = SampleResponse(id = output.id, name = output.name)
```

---

# Global Exception Handler

O tratamento de exceções deve permanecer centralizado na camada Presentation.

Todos os erros da aplicação devem ser traduzidos para respostas HTTP padronizadas utilizando `ProblemDetail`, conforme definido pela **RFC 9457 (Problem Details for HTTP APIs)**.

As seguintes regras devem ser respeitadas:

- Toda exceção deve ser tratada pelo Global Exception Handler.
- Respostas de erro devem utilizar exclusivamente `ProblemDetail`.
- O código HTTP deve representar corretamente a natureza do erro.
- Exceções de infraestrutura não devem ser expostas ao cliente.
- Logs de exceção devem permanecer centralizados no Global Exception Handler.
- Todo handler loga em nível `ERROR` com `logger.error("...: ${ex.message}", ex)`.
- Informações sensíveis não devem ser incluídas na resposta.
- O formato das respostas de erro deve ser consistente em toda a aplicação.
- Nenhum controller captura exceções diretamente.

## ✔ Correto

```kotlin
@RestControllerAdvice
class GlobalExceptionHandler {

    @ExceptionHandler(Exception::class)
    fun handleGeneric(ex: Exception): ProblemDetail {
        logger.error("Unexpected error: ${ex.message}", ex)

        return buildProblemDetail(
            HttpStatus.INTERNAL_SERVER_ERROR,
            "Internal server error"
        )
    }

    private fun buildProblemDetail(status: HttpStatus, message: String): ProblemDetail =
        ProblemDetail.forStatusAndDetail(
            HttpStatusCode.valueOf(status.value()),
            message
        )
}
```

---

# Resolvers

Resolvers devem encapsular a obtenção de informações do contexto da requisição.

`AuthenticatedUserResolver` é o único ponto de extração do ID do usuário autenticado a partir do `SecurityContext`.

- `getUserId()` — lança exceção se não autenticado. Use em todos os endpoints protegidos.
- `getUserIdOrNull()` — retorna `null` se não autenticado. Use apenas em endpoints opcionalmente autenticados.

O principal no `SecurityContext` é sempre uma `String` com o UUID do usuário.

---

# Documentação

Toda a documentação da API é declarada fora dos controllers, em interfaces e classes dedicadas dentro de `presentation/controller/documentation/`. Os controllers permanecem limpos — sem nenhuma anotação do SpringDoc.

## `*ControllerDoc` — documentação de endpoints

- Cada controller possui uma interface `*ControllerDoc` correspondente em `documentation/`.
- A interface é anotada com `@Tag(name = "...", description = "...")` e `@SecurityRequirement` quando aplicável. **Não** declara `@RequestMapping` — o path fica exclusivamente no controller.
- Endpoints que exigem autenticação recebem `@SecurityRequirement(name = "bearerAuth")` na interface (ou no método, quando apenas alguns endpoints do controller são protegidos).
- Cada método da interface declara exatamente uma anotação `@Operation` e uma `@ApiResponses`.

**`@Operation`** deve conter:
- `summary`: título curto do endpoint (ex: `"Criar nova tarefa"`).
- `description`: descrição em HTML construída por concatenação de strings (`"<p>...</p>" + "<ul>..."`) com tags `<p>`, `<ul>`, `<li>`, `<code>` e `<blockquote>` quando necessário. Nunca use text blocks (triple-quote `"""`), pois não são suportados neste contexto.
- `requestBody` (quando aplicável): com `mediaType = "application/json"`, `schema` apontando para a classe de request e ao menos dois `ExampleObject` — um válido e um inválido.
- `parameters` (quando aplicável): para path variables, com `name`, `description`, `required = true` e `example`.

**`@ApiResponses`** deve cobrir todos os status HTTP possíveis para o endpoint:
- Sucesso (`2xx`): com `schema` e ao menos um `ExampleObject` com corpo representativo.
- Erros de validação (`400`): quando o endpoint aceita `@RequestBody`.
- Não autenticado (`401`): em todo endpoint protegido.
- Acesso negado (`403`): quando há verificação de autorização além da autenticação, por exemplo verificação de ownership, checagem de papel/permissão (role/permission) ou regra de negócio que restringe o acesso ao recurso.
- Não encontrado (`404`): quando o use case pode lançar `*NotFoundException`.
- Conflito (`409`): quando há verificação de unicidade.
- Erro interno (`500`): sempre presente, com exemplo padrão.

Para respostas sem corpo (`204 No Content`), omita o `content` na `@ApiResponse`.

### ✔ Correto

```kotlin
@Tag(name = "Samples", description = "Gerenciamento de samples — criação, listagem, atualização, exclusão e conclusão")
@SecurityRequirement(name = "bearerAuth")
interface SampleControllerDoc {

    @Operation(
        summary = "Criar novo sample",
        description = "<p>Cria um novo sample.</p>" +
            "<p>Requer autenticação via <code>Authorization: Bearer &lt;accessToken&gt;</code>.</p>",
        requestBody = RequestBody(
            required = true,
            content = [Content(
                mediaType = "application/json",
                schema = Schema(implementation = CreateSampleRequest::class),
                examples = [
                    ExampleObject(name = "Sample válido", value = """{"name": "Sample Name"}"""),
                    ExampleObject(name = "Nome em branco (inválido)", value = """{"name": ""}""")
                ]
            )]
        )
    )
    @ApiResponses(
        ApiResponse(
            responseCode = "201",
            description = "Sample criado com sucesso",
            content = [Content(
                mediaType = "application/json",
                schema = Schema(implementation = SampleResponse::class),
                examples = [ExampleObject(name = "Sample criado", value = """{"id": "b2c3d4e5-f6a7-8901-bcde-f12345678901", "name": "Sample Name"}""")]
            )]
        ),
        ApiResponse(
            responseCode = "400",
            description = "Dados de entrada inválidos",
            content = [Content(
                mediaType = "application/json",
                examples = [ExampleObject(name = "Validação falhou", value = """{"status": 400, "detail": "Validation failed", "title": "Bad Request"}""")]
            )]
        ),
        ApiResponse(
            responseCode = "401",
            description = "Token ausente, inválido ou expirado",
            content = [Content(
                mediaType = "application/json",
                examples = [ExampleObject(name = "Não autenticado", value = """{"status": 401, "detail": "Invalid or expired token"}""")]
            )]
        ),
        ApiResponse(
            responseCode = "500",
            description = "Erro interno inesperado",
            content = [Content(
                mediaType = "application/json",
                examples = [ExampleObject(name = "Erro interno", value = """{"status": 500, "detail": "Internal server error"}""")]
            )]
        )
    )
    fun createSample(@Valid @RequestBody request: CreateSampleRequest): ResponseEntity<SampleResponse>
}
```

### ✔ Correto — Controller implementando a interface Doc

```kotlin
@RestController
@RequestMapping("/api/v1/samples")
class SampleController(
    private val createSampleUseCase: CreateSampleUseCase
) : SampleControllerDoc {

    override fun createSample(@Valid @RequestBody request: CreateSampleRequest): ResponseEntity<SampleResponse> {
        logger.info("Creating sample - request: $request")

        val input = request.toInputDto()
        val output = createSampleUseCase.execute(input)

        val response = SampleResponse.of(output)

        logger.info("Creating sample - response: $response")
        return ResponseEntity.status(HttpStatus.CREATED).body(response)
    }

    companion object {
        private val logger = LoggerFactory.getLogger(SampleController::class.java)
    }
}
```

---

## `*RequestDoc` / `*ResponseDoc` — documentação de payloads

- Cada `data class` de request ou response implementa uma interface `*Doc` correspondente em `documentation/payload/{domínio}/request/` ou `.../response/`.
- A interface é anotada com `@Schema(name = "NomeDaClasse", description = "...")`.
- Cada propriedade da interface declara `@get:Schema` com `description`, `example` e, quando aplicável, `minLength` / `maxLength`.
- O `data class` de request/response implementa a interface — as anotações `@Schema` são herdadas automaticamente pelo SpringDoc.
- Campos sensíveis (senha, token) devem ter `example` com valor fictício (nunca omitir o exemplo).

### ✔ Correto — Interface de documentação de Request

```kotlin
@Schema(name = "CreateSampleRequest", description = "Dados para criação de um novo sample")
interface CreateSampleRequestDoc {

    @get:Schema(
        description = "Nome do sample. Não pode ser vazio e deve ter no máximo 255 caracteres.",
        example = "Sample Name",
        maxLength = 255
    )
    val name: String
}
```

### ✔ Correto — Data class implementando a interface Doc

```kotlin
data class CreateSampleRequest(
    @field:NotBlank
    @field:Size(max = 255)
    override val name: String
) : CreateSampleRequestDoc
```

### ✔ Correto — Interface de documentação de Response

```kotlin
@Schema(name = "SampleResponse", description = "Dados de um sample")
interface SampleResponseDoc {

    @get:Schema(
        description = "Identificador único do sample.",
        example = "b2c3d4e5-f6a7-8901-bcde-f12345678901"
    )
    val id: String

    @get:Schema(
        description = "Nome do sample.",
        example = "Sample Name"
    )
    val name: String
}
```

### ✔ Correto — Data class de Response com factory estática

```kotlin
data class SampleResponse(
    override val id: String,
    override val name: String
) : SampleResponseDoc {

    companion object {
        fun of(output: SampleOutputDto): SampleResponse =
            SampleResponse(
                id = output.id,
                name = output.name
            )
    }
}
```

### ✔ Correto — Interface Doc com campo sensível

```kotlin
@Schema(name = "SampleLoginRequest", description = "Credenciais para autenticação")
interface SampleLoginRequestDoc {

    @get:Schema(
        description = "Nome de usuário. Deve ter entre 3 e 50 caracteres alfanuméricos ou underscore.",
        example = "sample_user",
        minLength = 3,
        maxLength = 50
    )
    val username: String

    @get:Schema(
        description = "Senha. Deve ter entre 8 e 128 caracteres.",
        example = "S3cur3P@ss",
        minLength = 8,
        maxLength = 128
    )
    val password: String
}
```

### ✔ Correto — Request com toString protegido

```kotlin
data class SampleLoginRequest(
    @field:NotBlank
    @field:Size(min = 3, max = 50)
    override val username: String,
    @field:NotBlank
    @field:Size(min = 8, max = 128)
    override val password: String
) : SampleLoginRequestDoc {

    override fun toString(): String =
        "SampleLoginRequest{username='$username', password='[PROTECTED]'}"
}
```

### ✔ Correto — Response com tokens protegidos

```kotlin
@Schema(name = "SampleLoginResponse", description = "Tokens de autenticação")
interface SampleLoginResponseDoc {

    @get:Schema(
        description = "Token de acesso JWT.",
        example = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
    )
    val accessToken: String

    @get:Schema(
        description = "Token de refresh JWT.",
        example = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
    )
    val refreshToken: String
}

data class SampleLoginResponse(
    override val accessToken: String,
    override val refreshToken: String
) : SampleLoginResponseDoc {

    override fun toString(): String =
        "SampleLoginResponse{accessToken='[PROTECTED]', refreshToken='[PROTECTED]'}"

    companion object {
        fun of(output: SampleLoginOutputDto): SampleLoginResponse =
            SampleLoginResponse(
                accessToken = output.accessToken,
                refreshToken = output.refreshToken
            )
    }
}
```

---

## Regras gerais de documentação

- Nunca adicione anotações do SpringDoc (`@Operation`, `@ApiResponse`, `@Schema`, etc.) diretamente nos controllers ou nas `data class` de request/response. Toda documentação pertence às interfaces `*Doc`.
- Exemplos de corpo de resposta de erro devem seguir o formato Problem Details (RFC 9457, que obsoletou a RFC 7807): campos `type`, `title`, `status`, `detail`.
- Os nomes dos `@ExampleObject` devem ser descritivos e em português, indicando o cenário representado (ex: `"Sample válido"`, `"Nome em branco (inválido)"`).

---

# Resumo das Convenções

Além dos princípios transversais de `convention-layer-presentation.md`, o acesso HTTP deve respeitar:

- Controllers representam apenas endpoints HTTP.
- Controllers implementam `*ControllerDoc` para documentação (nenhuma anotação do SpringDoc no corpo do controller).
- Payloads representam apenas contratos HTTP.
- Requests e Responses devem utilizar `data class`.
- Requests possuem o método `toInputDto()`; Responses possuem a factory `of(output)` no companion object.
- Requests/Responses implementam interfaces `*Doc` para documentação.
- Toda exceção deve ser tratada pelo Global Exception Handler, respondendo em `ProblemDetail` (RFC 9457).
- Documentação OpenAPI deve permanecer desacoplada da implementação, nas interfaces `*Doc`.
