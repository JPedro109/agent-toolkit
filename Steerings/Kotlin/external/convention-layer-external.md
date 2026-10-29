# Convenções da Camada External

A camada **External** é responsável por implementar todas as integrações com tecnologias externas utilizadas pela aplicação.

Seu objetivo é isolar detalhes de infraestrutura da camada de domínio, mantendo a arquitetura desacoplada e seguindo o princípio da inversão de dependência.

Toda dependência de frameworks, bibliotecas, serviços externos ou infraestrutura deve permanecer exclusivamente nesta camada.

**Exceção — camada de apresentação (HTTP, scheduler)**: mecanismos de entrada da aplicação não são infraestrutura de suporte e por isso **não** pertencem a `external`. Tudo que representa uma forma de acesso à aplicação pertence à camada `presentation`.

---

# Responsabilidades

A camada External é responsável por:

- Implementar portas definidas pela camada Application.
- Integrar com bibliotecas externas.
- Integrar com serviços externos.
- Integrar com mecanismos de autenticação e autorização.
- Integrar com serviços de mensageria.
- Integrar com serviços de armazenamento.
- Integrar com serviços de cache.
- Adaptar modelos da aplicação para tecnologias externas.

A camada External nunca deve implementar regras de negócio.

---

# Fluxo de Dependências

A camada External depende da camada Application para implementar suas portas.

```text
Presentation
        ↓
Application
        ↑
External
        ↓
Frameworks / Banco / APIs / Bibliotecas
```

A camada Domain nunca possui dependência da camada External.

---

# Adaptadores

Todo componente da camada External deve representar um adaptador entre a aplicação e uma tecnologia externa.

Cada adaptador **obrigatoriamente** implementa uma interface de porta definida em `application/port/`. Nunca crie uma classe de infraestrutura sem uma interface correspondente em `port/`.

Sempre que uma nova integração for necessária, o fluxo deve ser:

1. Definir a porta na camada Application.
2. Implementar a porta na camada External.
3. Registrar o adaptador como Bean do Spring (`@Repository`, `@Component`).
4. Criar testes para o adaptador.

## ✔ Correto

```kotlin
interface SampleGateway {
    fun send(request: SampleRequest): SampleResponse
}
```

```kotlin
@Component
class SampleGatewayAdapter(
    private val client: SampleClient
) : SampleGateway {

    override fun send(request: SampleRequest): SampleResponse {
        ...
    }
}
```

---

# Organização

Os componentes devem ser organizados conforme sua responsabilidade.

Exemplo:

```text
external/
├── persistence/
├── security/
├── messaging/
├── storage/
├── cache/
└── client/
```

Novos módulos podem ser criados sempre que uma nova responsabilidade surgir.

---

# Configurações

A configuração de um adaptador específico desta camada — por exemplo, as propriedades de um cliente HTTP de integração — reside junto do adaptador, na própria External.

Já a configuração técnica de bootstrap transversal da aplicação (segurança, JWT, documentação/Swagger, tracing) reside no pacote transversal `configuration/`, fora de `core/`, conforme definido no documento de arquitetura. Assim, por exemplo, `SecurityConfigProperties` (propriedades de JWT/segurança) vive em `configuration/security/`, e não na External.

As seguintes regras devem ser respeitadas:

- Nunca acessar propriedades diretamente com `@Value`.
- Utilizar `@ConfigurationProperties`.
- Centralizar configurações por responsabilidade.
- Classes de propriedades devem possuir o sufixo `ConfigProperties`.
- Não implementar regras de negócio.

## ✔ Correto

```kotlin
@ConfigurationProperties(prefix = "sample.client")
data class SampleClientConfigProperties(
    val url: String,
    val timeout: Duration
)
```

---

# Exceções

Toda exceção proveniente de bibliotecas externas e que tem valor para as regras de negócio do domínio deve ser traduzida antes de atravessar os limites da camada External as demais exceções não precisam ser traduzidas.

A camada Domain nunca deve conhecer exceções de bibliotecas ou frameworks.

## ✔ Correto

```kotlin
runCatching {

}.getOrElse {
    logger.error("Invalid lib exception ${it.message}", it)

    throw InvalidLibException()
}
```

---

# Resumo das Convenções

- Toda integração externa pertence à camada External.
- Todo adaptador deve implementar uma porta da camada Application.
- Adaptadores usam sufixo `Adapter` (ex: `TokenAdapter`, `SampleRepositoryAdapter`).
- A camada External nunca implementa regras de negócio.
- Toda exceção de infraestrutura deve ser traduzida.
- Configurações devem utilizar `@ConfigurationProperties`.
- Cada responsabilidade deve possuir seu próprio pacote.
- Toda implementação de persistência deve seguir o documento de Convenções de Persistência.
