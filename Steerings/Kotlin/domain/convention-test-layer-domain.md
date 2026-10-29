# Testes Unitários de Value Objects

Os testes de **Value Objects** têm como objetivo garantir que todas as regras de validação e invariantes do objeto sejam respeitadas.

Cada Value Object deve possuir uma classe de teste dedicada, validando tanto cenários válidos quanto inválidos.

---

## Estrutura

Os testes devem permanecer na mesma organização lógica do Value Object.

Exemplo:

```text
src/test/kotlin/
└── core/
    └── domain/
        └── sample/
            └── valueobject/
                └── SampleNameValueObjectTest.kt
```

---

## Convenções

As seguintes regras devem ser respeitadas:

- Cada Value Object deve possuir sua própria classe de teste.
- Todos os métodos públicos devem ser testados.
- Devem ser testados cenários de sucesso e de falha.
- Testes devem validar o objeto retornado e não apenas a ausência de exceções.
- Quando o Value Object utilizar `Result<T>`, devem ser testados tanto o caminho de sucesso (`Result.success`, verificável por `isFailure = false`) quanto o de falha (`Result.fail`, verificável por `isFailure = true`), validando o valor retornado no sucesso e o tipo de exceção no erro.
- Fixtures devem ser utilizadas sempre que houver reutilização de dados de teste. Diferente das entidades, aqui a fixture **não é obrigatória**: ela entra quando um mesmo dado válido se repete entre cenários. Os valores de borda — nulo, vazio, no limite, acima do limite — devem permanecer **inline** no teste, pois são o próprio foco da verificação; escondê-los atrás de uma fixture só afasta o leitor do dado que importa.
- Sem contexto Spring, sem MockK.

---

## Cenários obrigatórios

Todo Value Object deve possuir, no mínimo, testes para:

- Criação com dados válidos.
- Criação com dados inválidos.
- Valores vazios (quando aplicável).
- Valores em branco (quando aplicável).
- Valores fora dos limites definidos.
- Valores exatamente nos limites permitidos.

---

## ✔ Correto

```kotlin
class SampleNameValueObjectTest {

    @Test
    fun `should create a valid SampleNameValueObject`() {
        val name = "Sample Name"

        val result = SampleNameValueObject.of(name)

        assertThat(result.isFailure).isFalse()
        assertThat(result.getValueResult().asString()).isEqualTo(name)
    }

    @Test
    fun `should fail when sample name is blank`() {
        val result = SampleNameValueObject.of("")

        assertThat(result.isFailure).isTrue()
    }

    @Test
    fun `should fail when sample name exceeds max length`() {
        val longName = "a".repeat(256)

        val result = SampleNameValueObject.of(longName)

        assertThat(result.isFailure).isTrue()
    }
}
```

---

## ❌ Incorreto

```kotlin
@Test
fun `should create`() {
    SampleNameValueObject.of("Sample")
}
```

---

## ❌ Incorreto

```kotlin
@Test
fun `should throw exception`() {
    assertThrows<Exception> { SampleNameValueObject.of("") }
}
```

---

## Boas práticas

Sempre que possível:

- Um cenário por teste.
- Utilizar nomes descritivos com backticks.
- Validar explicitamente o conteúdo do `Result`.
- Evitar múltiplos cenários no mesmo teste.
- Evitar dependência entre testes.
- Manter os testes rápidos e determinísticos.

---

# Testes Unitários de Entidades

Os testes de **Entidades** têm como objetivo garantir que as regras de negócio implementadas na camada Domain sejam executadas corretamente.

Cada entidade deve possuir uma classe de teste dedicada, validando a criação do objeto, suas invariantes e todos os comportamentos expostos publicamente.

As entidades devem ser testadas isoladamente, sem dependências de infraestrutura, banco de dados ou contexto do Spring.

---

## Estrutura

Os testes devem permanecer organizados conforme a entidade.

Exemplo:

```text
src/test/kotlin/
└── core/
    └── domain/
        └── sample/
            └── SampleEntityTest.kt
```

---

## Convenções

As seguintes regras devem ser respeitadas:

- Cada entidade deve possuir sua própria classe de teste.
- Todos os métodos públicos devem ser testados.
- Todos os comportamentos da entidade devem ser validados.
- Toda regra de negócio implementada na entidade deve possuir testes.
- Utilizar Fixtures (`aSample()`) para a construção dos objetos nos testes de comportamento, invariantes e mudança de estado — ali a construção é apenas andaime e a fixture remove ruído repetido.
- Exceção: nos testes de **criação** (com sucesso e com dados inválidos), instanciar a entidade diretamente, pois o foco é o próprio construtor e a relação entrada→saída deve ficar visível. Use as constantes `DEFAULT_*` da fixture como entrada (nunca o `aSample()`, que esconderia o construtor; nunca literais crus, que espalhariam valores mágicos).
- Os testes devem utilizar AssertJ para todas as asserções.
- Cada teste deve validar um único cenário.
- Os testes devem seguir o padrão AAA (Arrange → Act → Assert).
- Sem contexto Spring, sem MockK.

---

## Cenários obrigatórios

Toda entidade deve possuir, no mínimo, testes para:

- Criação com dados válidos (happy path): verificar todos os campos via getters.
- Métodos de negócio: cada método de negócio deve ter ao menos um cenário de sucesso e um de falha.
- Criação com dados inválidos: IDs inválidos, campos obrigatórios nulos/vazios/fora do limite — assert em `DomainException`.

---

## ✔ Correto

```kotlin
class SampleEntityTest {

    @Test
    fun `should create sample when data is valid`() {
        val sample = SampleEntity(id = SampleFixture.DEFAULT_ID, name = SampleFixture.DEFAULT_NAME)

        val sampleId = sample.id
        val name = sample.name

        assertThat(sampleId.asString()).isEqualTo(SampleFixture.DEFAULT_ID)
        assertThat(name.asString()).isEqualTo(SampleFixture.DEFAULT_NAME)
    }

    @Test
    fun `should update sample name when new name is valid`() {
        val sample = SampleFixture.aSample()
        val newName = "Updated Sample Name"

        sample.updateName(newName)

        assertThat(sample.name.asString()).isEqualTo(newName)
    }

    @Test
    fun `should throw when sample name is blank`() {
        val emptyName = ""

        assertThatThrownBy {
            SampleEntity(id = SampleFixture.DEFAULT_ID, name = emptyName)
        }.isInstanceOf(DomainException::class.java)
    }
}
```

---

## ❌ Incorreto

```kotlin
@Test
fun `should create sample`() {
    SampleFixture.aSample()
}
```

---

## ❌ Incorreto

```kotlin
@SpringBootTest
class SampleEntityTest { }
```

---

## Boas práticas

Sempre que possível:

- Reutilizar Fixtures.
- Validar explicitamente o estado da entidade.
- Testar apenas comportamentos públicos.
- Não acessar atributos privados por reflexão.
- Evitar múltiplos cenários no mesmo teste.
- Utilizar nomes descritivos com backticks.
- Manter os testes independentes entre si.

---

## Resumo das Convenções

Todo teste de Entidade deve respeitar os seguintes princípios:

- Cada entidade possui sua própria classe de teste.
- Apenas a camada Domain deve ser testada.
- Usar a fixture `aSample()` nos testes de comportamento, invariantes e estado; nos testes de criação, instanciar direto usando as constantes `DEFAULT_*`.
- Utilizar AssertJ para todas as validações.
- Usar backticks para nomes de testes.
- Um cenário por teste.
- Todos os comportamentos públicos devem ser cobertos.
- Os testes devem ser rápidos, determinísticos e independentes.
