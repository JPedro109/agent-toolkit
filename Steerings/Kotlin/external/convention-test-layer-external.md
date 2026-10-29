# Testes de External

Os testes de integração de adaptadores têm como objetivo validar o comportamento dos componentes da camada **External** que integram a aplicação com bibliotecas, frameworks ou serviços externos.

Diferentemente dos testes de persistência, esses testes não validam acesso ao banco de dados. Seu objetivo é garantir que a implementação concreta do adaptador funcione corretamente utilizando suas dependências reais.

---

## Estrutura

Esses testes devem permanecer organizados conforme o componente testado.

Exemplo:

```text
src/test/kotlin/
└── core/
    └── external/
        └── security/
            ├── TokenAdapterTest.kt
            └── PasswordEncoderAdapterTest.kt
```

---

## Convenções

As seguintes regras devem ser respeitadas:

- Cada adaptador deve possuir sua própria classe de teste.
- Os adaptadores devem ser instanciados manualmente no `@BeforeEach`.
- Não deve ser utilizado contexto Spring.
- Não devem ser utilizados mocks (MockK) das bibliotecas externas.
- Devem ser utilizadas implementações reais das dependências externas.
- Configurações necessárias para os testes devem ser criadas manualmente (ex: `SecurityConfigProperties` construído programaticamente).
- Constantes utilizadas durante os testes devem ser declaradas como `private const val` no `companion object`.
- Usar backticks para nomes de testes.
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

---

## ✔ Correto

```kotlin
class SampleAdapterTest {

    private lateinit var adapter: SampleAdapter

    @BeforeEach
    fun setUp() {
        val properties = SampleProperties(SECRET)

        adapter = SampleAdapter(properties)
    }

    @Test
    fun `should execute operation successfully`() {
        val sampleName = "sample"

        val result = adapter.execute(sampleName)

        assertThat(result).isNotNull()
    }

    companion object {
        private const val SECRET = "sample-secret-key"
    }
}
```

---

## ❌ Incorreto

```kotlin
@SpringBootTest
class SampleAdapterTest { }
```

---

## ❌ Incorreto

```kotlin
@ExtendWith(MockKExtension::class)
class SampleAdapterTest { }
```

---

## ❌ Incorreto

```kotlin
@MockK
private lateinit var jwtLibrary: JwtParser
```

---

## Boas práticas

Sempre que possível:

- Instanciar manualmente o componente em teste.
- Utilizar implementações reais das bibliotecas externas.
- Validar os cenários de sucesso e de falha.
- Um cenário por teste.
- Evitar dependência entre testes.
- Nomear os testes com backticks seguindo o padrão `should [resultado esperado] when [condição]`.

---

## Resumo das Convenções

Todo teste de integração de adaptadores deve respeitar os seguintes princípios:

- Cada adaptador possui sua própria classe de teste.
- Não utilizar contexto Spring.
- Não utilizar MockK.
- Utilizar implementações reais das dependências externas.
- Instanciar manualmente o componente em teste.
- Utilizar AssertJ para todas as validações.
- Os testes devem ser rápidos, determinísticos e independentes.
