# Convenções da Camada External — Gateway (HTTP)

Este documento detalha a integração com **outras aplicações via HTTP** (gateways). Ele é uma extensão do documento base `convention-layer-external.md` e assume **todos** os princípios da camada External definidos lá (todo adaptador implementa uma porta da Application, nunca contém regra de negócio, traduz exceções de infraestrutura e centraliza configuração via `@ConfigurationProperties`).

Um gateway é o adaptador que fala com um serviço HTTP remoto. Para o resto da aplicação ele é invisível: o caso de uso enxerga apenas a **porta** (um contrato em termos do domínio); quem traduz essa porta em chamadas HTTP é o gateway, nos limites da External.

---

# Princípios

- **Sempre há uma porta para implementar.** Nenhum gateway existe sem uma interface correspondente em `application/port/gateway/`. O caso de uso depende da porta, nunca do gateway concreto nem do client HTTP.
- **A porta fala a língua do domínio.** A assinatura da porta recebe e **retorna exclusivamente tipos do domínio** (`Entity`, `ValueObject`) ou tipos primitivos — nunca os DTOs HTTP da integração. Converter o payload remoto no tipo do domínio é responsabilidade do gateway.
- **Feign Client para o transporte.** A chamada HTTP é feita por uma interface **Feign Client**, que declara os endpoints remotos e trafega os DTOs da integração. O Feign client nunca é exposto fora da External.
- **Resiliência no adaptador.** Circuit breaker e retry são aplicados via **Resilience4j** (`@CircuitBreaker` e `@Retry`) no método do gateway que faz a chamada remota.
- **Tradução de falha.** Erros do client (HTTP, timeout, circuito aberto) que tenham valor de negócio são traduzidos para uma exceção do domínio/aplicação antes de cruzar a fronteira da External.

---

# Estrutura

O gateway reside em `external/gateway/`, com o Feign client e os DTOs da integração em subpacotes próprios. A porta vive na Application.

```text
application/
└── port/
    └── gateway/
        └── SampleGateway.java        # porta — fala tipos do domínio

external/
└── gateway/
    └── sample/
        ├── SampleGatewayAdapter.java # implementa a porta, resiliência + tradução
        ├── client/
        │   └── SampleFeignClient.java # interface Feign — fala HTTP
        └── dto/
            └── SampleClientResponse.java
```

---

# Porta

A porta é definida na camada Application e expressa o comportamento em termos do domínio.

```java
public interface SampleGateway {

    SampleEntity fetchById(IdValueObject id);
}
```

> A porta recebe `IdValueObject` e devolve `SampleEntity` — tipos do domínio. Ela não conhece HTTP, Feign nem os DTOs da integração.

---

# Feign Client

O Feign client é uma interface anotada com `@FeignClient` que declara os endpoints do serviço remoto. Ele trafega os **DTOs da integração**, nunca tipos do domínio.

```java
@FeignClient(
        name = "sample",
        url = "${sample.gateway.url}"
)
public interface SampleFeignClient {

    @GetMapping("/samples/{id}")
    SampleClientResponse getById(@PathVariable("id") String id);
}
```

Os DTOs da integração representam exatamente o contrato do serviço remoto e vivem na External, junto do client:

```java
public record SampleClientResponse(
        String id,
        String name
) { }
```

---

# Gateway Adapter

O adaptador implementa a porta, orquestra a chamada ao Feign client, aplica a resiliência com Resilience4j e traduz o resultado (e as falhas) para o domínio.

```java
@Component
public class SampleGatewayAdapter implements SampleGateway {

    private static final Logger LOGGER = LoggerFactory.getLogger(SampleGatewayAdapter.class);

    private final SampleFeignClient client;

    public SampleGatewayAdapter(SampleFeignClient client) {
        this.client = client;
    }

    @Override
    @CircuitBreaker(name = "sampleGateway", fallbackMethod = "fetchByIdFallback")
    @Retry(name = "sampleGateway")
    public SampleEntity fetchById(IdValueObject id) {
        var response = client.getById(id.asString());

        return new SampleEntity(response.id(), response.name());
    }

    private SampleEntity fetchByIdFallback(IdValueObject id, Throwable ex) {
        LOGGER.error("Sample gateway failure for id={}: {}", id.asString(), ex.getMessage(), ex);

        throw new SampleUnavailableException();
    }
}
```

As seguintes regras devem ser respeitadas:

- O adaptador usa o sufixo `Adapter` e é anotado com `@Component`.
- O método público implementa a porta (`@Override`) e devolve o tipo do domínio.
- A conversão do DTO da integração para o tipo do domínio ocorre aqui — nunca vaza `SampleClientResponse` para fora.
- `@CircuitBreaker` e `@Retry` do Resilience4j são aplicados no método que faz a chamada remota, com o mesmo `name` apontando para a instância configurada.
- O `fallbackMethod` tem a mesma assinatura do método, acrescida de um parâmetro `Throwable`, e traduz a falha para uma exceção com valor de negócio.

## ❌ Incorreto — retornar o DTO da integração em vez do tipo do domínio

```java
@Component
public class SampleGatewayAdapter implements SampleGateway {

    @Override
    public SampleClientResponse fetchById(IdValueObject id) {
        return client.getById(id.asString());
    }
}
```

## ❌ Incorreto — caso de uso dependendo do Feign client (sem porta)

```java
@Service
public class GetSampleUseCase {

    private final SampleFeignClient client;

    public SampleOutputDto execute(GetSampleInputDto input) {
        var response = client.getById(input.id());
        ...
    }
}
```

---

# Configuração

A habilitação do Feign (`@EnableFeignClients`) é configuração de bootstrap e reside no pacote transversal `configuration/`.

As propriedades de cada gateway (URL, timeout) ficam em uma classe `*ConfigProperties` com `@ConfigurationProperties`, conforme o documento base da External.

```java
@ConfigurationProperties(prefix = "sample.gateway")
public record SampleGatewayConfigProperties(
        String url,
        Duration timeout
) { }
```

## Timeouts do Feign

Todo gateway **deve** definir timeouts explícitos. O default do Feign/cliente HTTP subjacente costuma ser "sem limite" ou muito alto, e uma chamada pendurada é pior do que uma que falha: ela prende a thread, estoura o pool de conexões e — crucialmente — **nunca dispara o retry nem o circuit breaker**, que só reagem a falhas. O timeout é o que converte "travado" em "falhou", permitindo que a resiliência atue.

São dois timeouts distintos:

- **`connect-timeout`** — tempo máximo para **estabelecer a conexão TCP** com o host remoto. Expira quando o destino está inacessível, a rede caiu ou o host não responde ao handshake. Costuma ser curto (poucos segundos): se não conectou rápido, dificilmente conectará.
- **`read-timeout`** (o "gateway timeout" da chamada) — tempo máximo de espera pela **resposta depois de a conexão estar aberta**. Expira quando o serviço remoto aceitou a conexão mas demora a responder (processamento lento, dependência travada). Deve refletir o tempo aceitável de resposta do endpoint.

Configuração por `@FeignClient` (via propriedade, referenciando o `name` do client):

```yaml
spring:
  cloud:
    openfeign:
      client:
        config:
          sample:            # mesmo 'name' do @FeignClient
            connect-timeout: 2000   # ms para abrir a conexão
            read-timeout: 5000      # ms para receber a resposta
```

> O `read-timeout` deve ser **coerente com o Resilience4j**: se o `slow-call-duration-threshold` do circuit breaker for menor que o `read-timeout`, chamadas lentas são classificadas como lentas antes de estourar o timeout; e o `read-timeout` precisa ser curto o bastante para que as retentativas do `@Retry` caibam dentro de um tempo total de resposta aceitável (`max-attempts` × `read-timeout` + backoff).

As instâncias do Resilience4j são configuradas por propriedade (`application.yml`), referenciadas pelo `name` usado nas anotações. As duas preocupações são distintas e complementares: o **retry** tenta superar uma falha transitória repetindo a chamada; o **circuit breaker** protege o sistema quando a falha deixa de ser transitória e vira sistêmica, parando de bater num serviço que já se sabe estar fora.

```yaml
resilience4j:
  circuitbreaker:
    instances:
      sampleGateway:
        sliding-window-type: COUNT_BASED
        sliding-window-size: 10
        minimum-number-of-calls: 5
        failure-rate-threshold: 50
        slow-call-duration-threshold: 2s
        slow-call-rate-threshold: 100
        wait-duration-in-open-state: 10s
        permitted-number-of-calls-in-half-open-state: 3
        automatic-transition-from-open-to-half-open-enabled: true
  retry:
    instances:
      sampleGateway:
        max-attempts: 3
        wait-duration: 500ms
        enable-exponential-backoff: true
        exponential-backoff-multiplier: 2
        retry-exceptions:
          - feign.FeignException.ServiceUnavailable
          - java.net.SocketTimeoutException
        ignore-exceptions:
          - feign.FeignException.BadRequest
```

## Circuit Breaker — propriedades e importância

- **`sliding-window-type`** (`COUNT_BASED` ou `TIME_BASED`): define se a janela que mede a taxa de falhas conta as **últimas N chamadas** ou os **últimos N segundos**. `COUNT_BASED` é o default e previsível para gateways de volume estável; `TIME_BASED` encaixa melhor em tráfego irregular.
- **`sliding-window-size`**: tamanho dessa janela (N chamadas ou N segundos). Janela maior suaviza o ruído de falhas isoladas, mas reage mais devagar; menor reage rápido, porém dispara com mais facilidade.
- **`minimum-number-of-calls`**: mínimo de chamadas registradas antes de o breaker **poder** calcular a taxa de falhas. Sem isso, 1 falha logo no início (1 de 1 = 100%) abriria o circuito indevidamente. Protege contra decisão estatística prematura.
- **`failure-rate-threshold`** (%): percentual de falhas na janela que faz o circuito **abrir**. É o principal botão de sensibilidade — 50 significa "abra se metade das chamadas estiver falhando".
- **`slow-call-duration-threshold`**: acima dessa duração, a chamada é contada como **lenta** (mesmo que tenha sucesso). Serve para reagir a um serviço que não caiu, mas degradou — latência alta costuma anteceder a queda.
- **`slow-call-rate-threshold`** (%): percentual de chamadas lentas que também abre o circuito. Complementa o threshold de falha com um de latência.
- **`wait-duration-in-open-state`**: quanto tempo o circuito fica **aberto** (rejeitando chamadas de imediato, acionando o fallback) antes de testar recuperação. Dá fôlego para o serviço remoto se restabelecer sem receber carga.
- **`permitted-number-of-calls-in-half-open-state`**: no estado **meio-aberto**, quantas chamadas de teste são liberadas para decidir se o serviço voltou. Poucas chamadas evitam reabrir sob um pico de recuperação.
- **`automatic-transition-from-open-to-half-open-enabled`**: se `true`, o breaker passa de aberto para meio-aberto sozinho ao fim da espera, sem precisar de uma chamada para disparar a transição.

## Retry — propriedades e importância

- **`max-attempts`**: número total de tentativas (inclui a primeira). O equilíbrio é fino: tentativas demais aumentam a latência percebida e podem amplificar a sobrecarga de um serviço já em dificuldade.
- **`wait-duration`**: espera entre tentativas. Sem ela, as retentativas chegam quase instantâneas e podem piorar o congestionamento.
- **`enable-exponential-backoff`** / **`exponential-backoff-multiplier`**: fazem a espera crescer a cada tentativa (500ms, 1s, 2s…). Essencial para não martelar um serviço instável em intervalos fixos; dá tempo crescente de recuperação.
- **`retry-exceptions`**: lista explícita das exceções que **justificam** retry — tipicamente transitórias (timeout, 503). Retentar o que não é transitório é desperdício.
- **`ignore-exceptions`**: exceções que **nunca** devem ser retentadas, como um `400 Bad Request`. Repetir uma requisição malformada só gera carga inútil, pois o resultado será idêntico.

> **Ordem importa.** O `@Retry` executa por dentro do `@CircuitBreaker`: as retentativas acontecem primeiro e, só então, o resultado (sucesso ou falha final) é contabilizado pelo breaker. Assim, falhas transitórias absorvidas pelo retry não contam para abrir o circuito — apenas as que persistem após todas as tentativas.

---

# Exceções

Falhas da integração que tenham valor de negócio devem ser traduzidas antes de cruzar a fronteira da External, conforme o documento base. Timeout, indisponibilidade e circuito aberto tornam-se uma exceção própria (ex.: `SampleUnavailableException`), tratada no `fallbackMethod`. A camada Domain e a Application nunca conhecem `FeignException` nem exceções do Resilience4j.

---

# Resumo das Convenções

Além dos princípios da camada External (`convention-layer-external.md`), todo gateway HTTP deve respeitar:

- Toda integração HTTP implementa uma porta definida em `application/port/gateway/`.
- A porta recebe e retorna exclusivamente tipos do domínio — nunca DTOs da integração.
- O transporte HTTP é feito por uma interface **Feign Client**, que trafega os DTOs da integração e nunca é exposta fora da External.
- O adaptador (sufixo `Adapter`, `@Component`) converte o payload remoto no tipo do domínio.
- Circuit breaker e retry usam **Resilience4j** (`@CircuitBreaker` + `@Retry`) no método do gateway, com `fallbackMethod` traduzindo a falha.
- Exceções de infraestrutura são traduzidas para exceções com valor de negócio antes de sair da External.
- Configuração via `@ConfigurationProperties` (`*ConfigProperties`) e instâncias do Resilience4j por propriedade.
