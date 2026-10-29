# Testes de External

Os testes de integração de adaptadores têm como objetivo validar o comportamento dos componentes da camada **External** que integram a aplicação com bibliotecas, frameworks ou serviços externos.

Diferentemente dos testes de persistência, esses testes não validam acesso ao banco de dados. Seu objetivo é garantir que a implementação concreta do adaptador funcione corretamente utilizando suas dependências reais.

Esses testes devem permanecer organizados conforme o componente testado.

Exemplo:

```text
src/test/java/
└── core/
    └── external/
        ├── security/
        │   ├── TokenAdapterTest.java
        │   └── PasswordEncoderAdapterTest.java
        ├── messaging/
        │   └── SamplePublisherAdapterTest.java
        └── storage/
            └── SampleStorageAdapterTest.java
```

---

## Convenções

As seguintes regras devem ser respeitadas:

- Cada adaptador deve possuir sua própria classe de teste.
- Os adaptadores devem ser instanciados manualmente.
- Não deve ser utilizado contexto Spring.
- Não devem ser utilizados mocks das bibliotecas externas.
- Devem ser utilizadas implementações reais das dependências externas.
- Configurações necessárias para os testes devem ser criadas manualmente.
- Constantes utilizadas durante os testes devem ser declaradas como `private static final`.
- Todo método deve possuir `@DisplayName`.
- Os testes devem utilizar AssertJ.
- Os testes devem seguir o padrão AAA (Arrange → Act → Assert).

---

## Cenários obrigatórios

Todo adaptador deve possuir testes para:

- Fluxo de sucesso.
- Cenários de erro.
- Valores inválidos.
- Valores limites.
- Comportamentos específicos da biblioteca integrada.
- Exceções esperadas.

Os cenários específicos devem refletir a responsabilidade do adaptador.

Exemplos:

- Adaptadores de Token:
    - geração de access token;
    - geração de refresh token;
    - validação com subject correto;
    - tokens expirados — devem lançar `InvalidTokenException`, nunca exceção da biblioteca;
    - tokens malformados — devem lançar `InvalidTokenException`;
    - tokens assinados com segredo diferente — devem lançar `InvalidTokenException`;
    - token do tipo errado (ex.: refresh token passado para `validateAccessToken`) — deve lançar `InvalidTokenException`.

- Adaptadores de PasswordEncoder:
    - encode retorna hash diferente do raw;
    - hashes distintos para a mesma senha;
    - `matches` retorna `true` para senha correta;
    - `matches` retorna `false` para senha errada.

- Adaptadores de mensageria:
    - envio da mensagem;
    - tratamento de falhas;
    - serialização.

---

## ✔ Correto

```java
class SampleAdapterTest {
    private static final String SECRET = "sample-secret-key";

    private SampleAdapter adapter;

    @BeforeEach
    void setUp() {
        var properties = new SampleProperties(SECRET);
        adapter = new SampleAdapter(properties);
    }

    @Test
    @DisplayName("Should execute operation successfully")
    void shouldExecuteOperationSuccessfully() {
        var sampleName = "sample";

        var result = adapter.execute(sampleName);

        assertThat(result).isNotNull();
    }

}
```

---

## ❌ Incorreto

```java
@SpringBootTest
class SampleAdapterTest { }
```

---

## ❌ Incorreto

```java
@ExtendWith(MockitoExtension.class)
class SampleAdapterTest { }
```

---

## ❌ Incorreto

```java
@Mock
private ExternalLibrary externalLibrary;
```

---

## Boas práticas

Sempre que possível:

- Instanciar manualmente o componente em teste.
- Utilizar implementações reais das bibliotecas externas.
- Validar os cenários de sucesso e de falha.
- Um cenário por teste.
- Evitar dependência entre testes.
- Nomear os testes seguindo o padrão `Should [resultado esperado] when [condição]`.

---

## Resumo das Convenções

Todo teste de integração de adaptadores deve respeitar os seguintes princípios:

- Cada adaptador possui sua própria classe de teste.
- Não utilizar contexto Spring.
- Não utilizar Mockito.
- Utilizar implementações reais das dependências externas.
- Instanciar manualmente o componente em teste.
- Utilizar AssertJ para todas as validações.
- Os testes devem ser rápidos, determinísticos e independentes.

