# Testes da Camada External — Gateway (HTTP)

Os testes de **Gateway** têm como objetivo garantir que o adaptador traduza corretamente a integração HTTP para o domínio: converter o payload remoto no tipo do domínio e traduzir falhas em exceções com valor de negócio.

Diferente dos demais adaptadores de External (Token, PasswordEncoder), que exigem a **dependência real** porque a biblioteca *é* o comportamento sob teste, o gateway é um **adaptador de orquestração**: o que lhe pertence é a conversão e a tradução de erro, não a mecânica HTTP do Feign. Por isso o teste é **unitário**, com o **Feign Client mockado** — o mesmo tratamento dado ao Repository Adapter (DAO mockado), em `convention-test-layer-external-persistence.md`.

> **Resiliência não é exercitada aqui.** As anotações `@CircuitBreaker` e `@Retry` do Resilience4j só são interceptadas pelos proxies do Spring. No teste unitário o adaptador é instanciado diretamente, então essas anotações ficam inertes — e tudo bem: circuit breaker e retry são comportamento do framework/configuração, não do código do adaptador (mesma lógica pela qual não se testa `@Scheduled`/`@SchedulerLock` no scheduler). O que se testa é o corpo do método e o `fallbackMethod`.

---

## Estrutura

Os testes devem permanecer organizados conforme o gateway.

Exemplo:

```text
src/test/java/
└── core/
    └── external/
        └── gateway/
            └── sample/
                └── SampleGatewayAdapterTest.java
```

---

## Convenções

As seguintes regras devem ser respeitadas:

- Cada Gateway Adapter deve possuir sua própria classe de teste.
- O teste é unitário: usar `@ExtendWith(MockitoExtension.class)`, declarar o Feign Client com `@Mock` e instanciar o adaptador com `@InjectMocks`.
- Nunca subir contexto Spring (`@SpringBootTest`): não há proxy a exercitar no corpo do adaptador.
- O Feign Client é mockado — nunca se faz chamada HTTP real nem se usa WireMock aqui.
- Deve ser validada a **conversão** do DTO da integração para o tipo do domínio (todos os campos).
- Deve ser validada a **tradução de falha**: quando o client lança, o adaptador propaga a exceção de negócio do `fallbackMethod`.
- Utilizar Fixtures para construção dos DTOs da integração e dos objetos do domínio.
- Os testes devem utilizar AssertJ e os verificadores do Mockito.
- Todo método deve possuir `@DisplayName`.
- Os testes devem seguir o padrão AAA (Arrange → Act → Assert).

---

## Dependências

Os testes de Gateway devem utilizar apenas:

- Mockito;
- Fixtures;
- AssertJ.

Não devem utilizar:

- Contexto Spring;
- Servidor HTTP real ou WireMock;
- Casos de uso reais.

---

## Cenários obrigatórios

Todo Gateway Adapter deve possuir, no mínimo, testes para:

- Sucesso: o client retorna o DTO e o adaptador devolve o tipo do domínio com os campos convertidos corretamente.
- Falha: o client lança (ex.: `FeignException`) e o adaptador traduz para a exceção de negócio do fallback.

---

## ✔ Correto

```java
@ExtendWith(MockitoExtension.class)
class SampleGatewayAdapterTest {

    @Mock
    private SampleFeignClient client;

    @InjectMocks
    private SampleGatewayAdapter adapter;

    @Test
    @DisplayName("Should return domain entity when client responds successfully")
    void shouldReturnDomainEntityWhenClientRespondsSuccessfully() {
        var id = IdValueObject.of(SampleFixture.DEFAULT_ID).getValueOrThrow();
        var response = new SampleClientResponse(SampleFixture.DEFAULT_ID, SampleFixture.DEFAULT_NAME);

        when(client.getById(SampleFixture.DEFAULT_ID)).thenReturn(response);

        var result = adapter.fetchById(id);

        assertThat(result.getId().asString()).isEqualTo(SampleFixture.DEFAULT_ID);
        assertThat(result.getName().asString()).isEqualTo(SampleFixture.DEFAULT_NAME);
        verify(client).getById(SampleFixture.DEFAULT_ID);
    }

    @Test
    @DisplayName("Should translate failure into business exception when client fails")
    void shouldTranslateFailureIntoBusinessExceptionWhenClientFails() {
        var id = IdValueObject.of(SampleFixture.DEFAULT_ID).getValueOrThrow();

        assertThatThrownBy(() -> adapter.fetchByIdFallback(id, new RuntimeException("boom")))
                .isInstanceOf(SampleUnavailableException.class);
    }
}
```

> O cenário de falha chama o `fallbackMethod` diretamente, porque é ele que contém a tradução. Como o Resilience4j não está ativo no teste unitário, o fallback não é invocado automaticamente — ele é verificado como o método que de fato é. Se o `fallbackMethod` for privado, torne-o package-private para permitir o teste, mantendo-o fora do contrato público.

---

## ❌ Incorreto — mockar o resultado em vez de testar a conversão

```java
when(adapter.fetchById(any())).thenReturn(SampleFixture.aSample());
```

## ❌ Incorreto — subir contexto Spring ou usar WireMock no teste unitário

```java
@SpringBootTest
class SampleGatewayAdapterTest { }
```

---

## Resumo das Convenções

Todo teste de Gateway Adapter deve respeitar os seguintes princípios:

- Cada gateway adapter possui sua própria classe de teste.
- O teste é unitário, com o Feign Client mockado via Mockito — sem contexto Spring, sem WireMock.
- Não se testa `@CircuitBreaker`/`@Retry` (responsabilidade do framework); testa-se a conversão e o `fallbackMethod`.
- Verifica-se a conversão do DTO da integração para o domínio e a tradução de falha em exceção de negócio.
- Usar `@ExtendWith(MockitoExtension.class)`, `@Mock` e `@InjectMocks`.
- Utilizar Fixtures e AssertJ.
- Todo método deve possuir `@DisplayName`.
- Um cenário por teste, seguindo o padrão AAA.
