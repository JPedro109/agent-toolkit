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
    ├── documentation/
    │   └── payload/
    ├── payload/
    │   ├── sample/
    │   │   ├── request/
    │   │   └── response/
    │   └── common/
    ├── common/
    │   ├── handler/
    │   └── resolver/
    └── SampleController.java
```

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
- Controllers devem depender exclusivamente de casos de uso.
- Controllers devem permanecer pequenos e objetivos.
- Controllers devem ter a factory do logger utilizando SLF4j utilizando uma constante estática privada.
- Controllers devem ter o log do início da requisição com a request se existir.
- Controllers devem ter o log do fim da requisição com a response se existir.
- Métodos dos controllers devem seguir estritamente a estrutura do exemplo.

## ✔ Correto

```java
@RestController
@RequestMapping("/samples")
public class SampleController {

    private static final Logger LOGGER = LoggerFactory.getLogger(SampleController.class);

    private final CreateSampleUseCase createSampleUseCase;

    public SampleController(
            CreateSampleUseCase createSampleUseCase
    ) {
        this.createSampleUseCase = createSampleUseCase;
    }

    @PostMapping
    public ResponseEntity<CreateSampleResponse> create(
            @Valid @RequestBody CreateSampleRequest request
    ) {

        LOGGER.info("Create sample request: {}", request);

        var dto = request.toInputDto();
        var output = createSampleUseCase.execute(dto);

        var response = CreateSampleResponse.of(output);

        LOGGER.info("Create sample response: {}", response);

        return ResponseEntity.ok(response);
    }
}
```

## ❌ Incorreto

```java
@RestController
public class SampleController {

    @Autowired
    private SampleRepository repository;
}
```

## ❌ Incorreto

```java
@PostMapping
public ResponseEntity<?> create(
        @RequestBody CreateSampleRequest request
) {

    if (repository.existsByName(request.name())) {
        throw new SampleAlreadyExistsException();
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

- Devem utilizar `record`.
- Devem representar exclusivamente o contrato HTTP.
- Devem ser imutáveis.
- Não devem possuir regras de negócio.
- Não devem possuir dependência da camada Domain.
- Devem utilizar Bean Validation para validações sintáticas e estruturais.
- Não devem conter lógica de conversão para objetos do domínio.
- Toda classe de Request deve possuir um método de instância `toInputDto` responsável por converter o payload HTTP para o Input DTO do caso de uso. A conversão é apenas cópia de campos — sem validação de negócio, sem transformação com regra (o espelho do `of` do Response, no sentido da entrada).
- Controllers não devem montar manualmente o Input DTO a partir do Request.

Validações apropriadas para Bean Validation incluem, por exemplo:

- Campos obrigatórios;
- Tamanho mínimo e máximo;
- Formato de e-mail;
- Expressões regulares;
- Valores mínimos e máximos;
- Validação de coleções.

### ✔ Correto

```java
public record CreateSampleRequest(
        @NotBlank
        @Size(max = 255)
        String name
) {

    public CreateSampleInputDto toInputDto() {
        return new CreateSampleInputDto(name);
    }
}
```

### ❌ Incorreto

```java
public record CreateSampleRequest(
        String name,
        String value
) {

    public CreateSampleRequest {
        if (name.startsWith("ADMIN")) {
            throw new SampleDomainException();
        }
    }
}
```

```java
public record CreateSampleRequest(
        SampleEntity sample
) { }
```

---

## Payloads de Response

Payloads de Response representam exclusivamente os dados retornados pela API.

As seguintes regras devem ser respeitadas:

- Devem utilizar `record`.
- Devem representar exclusivamente o contrato HTTP.
- Devem ser imutáveis.
- Não devem possuir regras de negócio.
- Não devem possuir dependência da camada Domain.
- Toda classe de Response deve possuir um método estático responsável por converter o DTO de saída do caso de uso para o payload HTTP.
- Controllers não devem realizar manualmente o mapeamento entre Output DTO e Response.

### ✔ Correto

```java
public record CreateSampleResponse(
        String id,
        String name
) {

    public static CreateSampleResponse of(
            CreateSampleOutputDto output
    ) {
        return new CreateSampleResponse(
                output.id(),
                output.name()
        );
    }
}
```

```java
@PostMapping
public ResponseEntity<CreateSampleResponse> create(
        @Valid @RequestBody CreateSampleRequest request
) {
    var dto = request.toInputDto();
    var output = createSampleUseCase.execute(dto);

    var response = CreateSampleResponse.of(output);

    return ResponseEntity.ok(response);
}
```

### ❌ Incorreto

```java
@PostMapping
public ResponseEntity<CreateSampleResponse> create(
        @RequestBody CreateSampleRequest request
) {

    var output = createSampleUseCase.execute(...);

    var response = new CreateSampleResponse(output.id(), output.name());

    return ResponseEntity.ok(response);
}
```

---

# Global Exception Handler

O tratamento de exceções deve permanecer centralizado na camada Presentation.

Todos os erros da aplicação devem ser traduzidos para respostas HTTP padronizadas utilizando a classe `ProblemDetail`, conforme definido pela **RFC 9457 (Problem Details for HTTP APIs)**.

As seguintes regras devem ser respeitadas:

- Toda exceção deve ser tratada pelo Global Exception Handler.
- Respostas de erro devem utilizar exclusivamente `ProblemDetail`, para contemplar a RFC 9457.
- O código HTTP deve representar corretamente a natureza do erro.
- Exceções de infraestrutura não devem ser expostas ao cliente.
- Logs de exceção devem permanecer centralizados no Global Exception Handler.
- Evitar registrar a mesma exceção em múltiplas camadas da aplicação.
- Informações sensíveis não devem ser incluídas na resposta.
- O formato das respostas de erro deve ser consistente em toda a aplicação.


## ✔ Correto

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    private static final Logger LOGGER = LoggerFactory.getLogger(GlobalExceptionHandler.class);

    @ExceptionHandler(Exception.class)
    public ProblemDetail handleGeneric(Exception ex) {
        LOGGER.error("Unexpected error: {}", ex.getMessage(), ex);

        return buildProblemDetail(HttpStatus.INTERNAL_SERVER_ERROR, "Internal server error");
    }

    private ProblemDetail buildProblemDetail(HttpStatus status, String message) {
        return ProblemDetail.forStatusAndDetail(HttpStatusCode.valueOf(status.value()), message);
    }
}
```

---

# Resolvers

Resolvers devem encapsular a obtenção de informações do contexto da requisição.

As seguintes regras devem ser respeitadas:

- Devem centralizar a leitura do contexto HTTP.
- Controllers não devem acessar diretamente objetos do framework quando houver um resolver apropriado.
- Devem possuir responsabilidade única.
- Por serem stateless e lerem apenas o contexto da requisição (ex.: `SecurityContextHolder`), devem ser classes utilitárias: `final`, com construtor privado e métodos estáticos. Não devem ser registrados como bean nem injetados.

## ✔ Correto

```java
public final class AuthenticatedUserResolver {

    private AuthenticatedUserResolver() { }

    public static String getUserId() {
        var authentication = SecurityContextHolder.getContext().getAuthentication();
        if (authentication == null) {
            throw new UnauthenticatedException("User is not authenticated");
        }

        return authentication.getPrincipal().toString();
    }
}
```

---

# Documentação

## `*ControllerDoc` — documentação de endpoints

- Cada controller possui uma interface `*ControllerDoc` correspondente em `documentation/` (ex: `SampleControllerDoc`).
- A interface é anotada com `@Tag(name = "...", description = "...")` e `@RequestMapping` com o path base do controller.
- Endpoints que exigem autenticação recebem `@SecurityRequirement(name = "bearerAuth")` na interface (ou no método, quando apenas alguns endpoints do controller são protegidos).
- Cada método da interface declara exatamente uma anotação `@Operation` e uma `@ApiResponses`.

**`@Operation`** deve conter:
- `summary`: título curto do endpoint (ex: `"Criar nova tarefa"`).
- `description`: descrição em HTML com `<p>`, `<ul>`, `<li>`, `<code>` e `<blockquote>` quando necessário. Sempre mencione o header de autenticação em endpoints protegidos.
- `requestBody` (quando aplicável): com `mediaType = "application/json"`, `schema` apontando para a classe de request e ao menos dois `@ExampleObject` — um válido e um inválido.
- `parameters` (quando aplicável): para path variables, com `name`, `description`, `required = true` e `example`.

**`@ApiResponses`** deve cobrir todos os status HTTP possíveis para o endpoint:
- Sucesso (`2xx`): com `schema` e ao menos um `@ExampleObject` com corpo representativo.
- Erros de validação (`400`): quando o endpoint aceita `@RequestBody`.
- Não autenticado (`401`): em todo endpoint protegido.
- Acesso negado (`403`): quando há verificação de autorização além da autenticação, por exemplo verificação de ownership, checagem de papel/permissão (role/permission) ou regra de negócio que restringe o acesso ao recurso.
- Não encontrado (`404`): quando o use case pode lançar `*NotFoundException`.
- Conflito (`409`): quando há verificação de unicidade.
- Erro interno (`500`): sempre presente, com exemplo padrão.

Para respostas sem corpo (`204 No Content`), omita o `content` na `@ApiResponse`.

```java
@Tag(name = "Samples", description = "Gerenciamento de samples — criação, listagem, atualização e exclusão")
@RequestMapping("/api/v1/samples")
@SecurityRequirement(name = "bearerAuth")
public interface SampleControllerDoc {

    @Operation(
            summary = "Criar novo sample",
            description = """
                    <p>Cria um novo sample associado ao contexto autenticado.</p>
                    <p>Requer autenticação via <code>Authorization: Bearer &lt;accessToken&gt;</code>.</p>
                    """,
            requestBody = @RequestBody(
                    required = true,
                    content = @Content(
                            mediaType = "application/json",
                            schema = @Schema(implementation = CreateSampleRequest.class),
                            examples = {
                                    @ExampleObject(name = "Sample válido", value = """
                                            {"name": "Sample Name"}
                                            """),
                                    @ExampleObject(name = "Nome em branco (inválido)", value = """
                                            {"name": ""}
                                            """)
                            }
                    )
            )
    )
    @ApiResponses({
            @ApiResponse(responseCode = "201", description = "Sample criado com sucesso",
                    content = @Content(mediaType = "application/json",
                            schema = @Schema(implementation = SampleResponse.class),
                            examples = @ExampleObject(name = "Sample criado", value = """
                                    {
                                      "id": "b2c3d4e5-f6a7-8901-bcde-f12345678901",
                                      "name": "Sample Name"
                                    }
                                    """))),
            @ApiResponse(responseCode = "401", description = "Token ausente, inválido ou expirado",
                    content = @Content(mediaType = "application/json",
                            examples = @ExampleObject(name = "Não autenticado", value = """
                                    {"status": 401, "detail": "Invalid or expired token"}
                                    """))),
            @ApiResponse(responseCode = "500", description = "Erro interno inesperado",
                    content = @Content(mediaType = "application/json",
                            examples = @ExampleObject(name = "Erro interno", value = """
                                    {"status": 500, "detail": "Internal server error"}
                                    """)))
    })
    ResponseEntity<SampleResponse> createSample(@Valid @RequestBody CreateSampleRequest request);
}
```

## `*RequestDoc` / `*ResponseDoc` — documentação de payloads

- Cada record de request ou response implementa uma interface `*Doc` correspondente em `documentation/payload/{domínio}/request/` ou `.../response/`.
- A interface é anotada com `@Schema(name = "NomeDaClasse", description = "...")`.
- Cada método da interface declara `@Schema` com `description`, `example` e, quando aplicável, `minLength` / `maxLength`.
- O record de request/response implementa a interface — as anotações `@Schema` dos métodos são herdadas automaticamente pelo SpringDoc.
- Campos sensíveis (senha, token) devem ter `example` com valor fictício (nunca omitir o exemplo).

```java
@Schema(name = "CreateSampleRequest", description = "Dados para criação de um novo sample")
public interface CreateSampleRequestDoc {

    @Schema(
            description = "Nome do sample. Não pode ser vazio e deve ter no máximo 255 caracteres.",
            example = "Sample Name",
            maxLength = 255
    )
    String name();
}

public record CreateSampleRequest(
        @NotBlank
        @Size(max = 255)
        String name
) implements CreateSampleRequestDoc { }
```

## Regras gerais de documentação

- Nunca adicione anotações do SpringDoc (`@Operation`, `@ApiResponse`, `@Schema`, etc.) diretamente nos controllers ou nos records de request/response. Toda documentação pertence às interfaces `*Doc`.
- Exemplos de corpo de resposta de erro devem seguir o formato Problem Details (RFC 9457, que obsoletou a RFC 7807): campos `type`, `title`, `status`, `detail`.
- Os nomes dos `@ExampleObject` devem ser descritivos e em português, indicando o cenário representado (ex: `"Sample válido"`, `"Nome muito curto (inválido)"`).

## ✔ Correto

```java
public interface SampleControllerDoc {

    @Operation(summary = "Create sample.")
    ResponseEntity<CreateSampleResponse> create(
            CreateSampleRequest request
    );

}
```

```java
@RestController
public class SampleController implements SampleControllerDoc {
    ...
}
```

---

# Resumo das Convenções

Além dos princípios transversais de `convention-layer-presentation.md`, o acesso HTTP deve respeitar:

- Controllers representam apenas endpoints HTTP.
- Payloads representam apenas contratos HTTP.
- Requests e Responses devem utilizar `record`.
- Requests possuem o método `toInputDto()`; Responses possuem a factory estática `of(output)`.
- Toda exceção deve ser tratada pelo Global Exception Handler, respondendo em `ProblemDetail` (RFC 9457).
- Documentação OpenAPI deve permanecer desacoplada da implementação, nas interfaces `*Doc`.
