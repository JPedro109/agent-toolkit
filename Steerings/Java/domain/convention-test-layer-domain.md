# Testes Unitários de Value Objects

Os testes de **Value Objects** têm como objetivo garantir que todas as regras de validação e invariantes do objeto sejam respeitadas.

Cada Value Object deve possuir uma classe de teste dedicada, validando tanto cenários válidos quanto inválidos.

---

## Estrutura

Os testes devem permanecer na mesma organização lógica do Value Object.

Exemplo:

```text
src/test/java/
└── core/
    └── domain/
        └── sample/
            └── valueobject/
                └── SampleNameValueObjectTest.java
```

---

## Convenções

As seguintes regras devem ser respeitadas:

- Cada Value Object deve possuir sua própria classe de teste.
- Todos os métodos públicos devem ser testados.
- Devem ser testados cenários de sucesso e de falha.
- Testes devem validar o objeto retornado e não apenas a ausência de exceções.
- Quando o Value Object utilizar `Result<T>`, devem ser testados tanto `Result.success(...)` quanto `Result.fail(...)`.
- Fixtures devem ser utilizadas sempre que houver reutilização de dados de teste. Diferente das entidades, aqui a fixture **não é obrigatória**: ela entra quando um mesmo dado válido se repete entre cenários. Os valores de borda — nulo, vazio, no limite, acima do limite — devem permanecer **inline** no teste, pois são o próprio foco da verificação; escondê-los atrás de uma fixture só afasta o leitor do dado que importa.

---

## Cenários obrigatórios

Todo Value Object deve possuir, no mínimo, testes para:

- Criação com dados válidos.
- Criação com dados inválidos.
- Valores nulos (quando aplicável).
- Valores vazios (quando aplicável).
- Valores fora dos limites definidos.
- Valores exatamente nos limites permitidos.

---

## ✔ Correto

```java
class SampleNameValueObjectTest {

    @Test
    @DisplayName("Should create a valid SampleNameValueObject")
    void shouldCreateValueObjectWhenValueIsValid() {
        var sampleName = "Sample";

        var result = SampleNameValueObject.of(sampleName);

        assertFalse(result.isFail());
        assertEquals(sampleName, result.getValue().asString());
    }

    @ParameterizedTest
    @DisplayName("Should fail when sample name is null or blank")
    @NullAndEmptySource
    void shouldFailForInvalidSampleName(String invalidSampleName) {
        var result = SampleNameValueObject.of(invalidSampleName);

        assertTrue(result.isFail());
        assertInstanceOf(DomainException.class, result.getError());
    }

}
```

---

## ✔ Correto

```java
@Test
@DisplayName("Should fail when sample name is null")
void shouldReturnFailureWhenValueIsNull() {
    String invalidSampleName = null;

    var result = SampleNameValueObject.of(invalidSampleName);

    assertTrue(result.isFail());
    assertInstanceOf(DomainException.class, result.getError());
}
```

---

## ❌ Incorreto

```java
@Test
void shouldCreate() {
    SampleNameValueObject.of("Sample");
}
```

---

## ❌ Incorreto

```java
@Test
void shouldThrowException() {
    assertThrows(Exception.class, () -> SampleNameValueObject.of(""));
}
```

---

## Boas práticas

Sempre que possível:

- Um cenário por teste.
- Utilizar nomes descritivos para os métodos de teste.
- Validar explicitamente o conteúdo do `Result`.
- Evitar múltiplos cenários no mesmo teste.
- Evitar dependência entre testes.
- Manter os testes rápidos e determinísticos.

---

## Resumo das Convenções

Todo teste de Value Object deve respeitar os seguintes princípios:

- Cada Value Object possui sua própria classe de teste.
- Cenários válidos e inválidos devem ser testados.
- Todo `Result.success(...)` e `Result.fail(...)` deve ser validado.
- Um cenário por teste.
- Testes devem validar explicitamente o resultado obtido.
- Fixtures são opcionais e entram apenas quando houver reutilização de dados; valores de borda ficam inline.

# Testes Unitários de Entidades

Os testes de **Entidades** têm como objetivo garantir que as regras de negócio implementadas na camada Domain sejam executadas corretamente.

Cada entidade deve possuir uma classe de teste dedicada, validando a criação do objeto, suas invariantes e todos os comportamentos expostos publicamente.

As entidades devem ser testadas isoladamente, sem dependências de infraestrutura, banco de dados ou contexto do Spring.

---

## Estrutura

Os testes devem permanecer organizados conforme a entidade.

Exemplo:

```text
src/test/java/
└── core/
    └── domain/
        └── sample/
            └── SampleEntityTest.java
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
- Todo método deve possuir `@DisplayName`.

---

## Cenários obrigatórios

Toda entidade deve possuir, no mínimo, testes para:

- Criação com dados válidos.
- Criação com dados inválidos.
- Execução dos comportamentos públicos.
- Alteração de estado.
- Validação das invariantes.
- Exceções lançadas pela entidade.

---

## ✔ Correto

```java
class SampleEntityTest {

    @Test
    @DisplayName("Should create sample when data is valid")
    void shouldCreateSampleWhenDataIsValid() {
        var sample = new SampleEntity(SampleFixture.DEFAULT_ID, SampleFixture.DEFAULT_NAME);

        var idValueObject = sample.getId();
        var nameValueObject = sample.getName();

        assertThat(idValueObject).isNotNull();
        assertThat(idValueObject.asString()).isEqualTo(SampleFixture.DEFAULT_ID);
        assertThat(nameValueObject.asString()).isEqualTo(SampleFixture.DEFAULT_NAME);
    }
}
```

---

## ✔ Correto

```java
@Test
@DisplayName("Should update name when new name is valid")
void shouldUpdateNameWhenNewNameIsValid() {
    var sample = SampleFixture.aSample();
    var newName = "Updated Sample";

    sample.updateName(newName);

    assertThat(sample.getName().asString()).isEqualTo(newName);
}
```

---

## ✔ Correto

```java
@Test
@DisplayName("Should throw when name is invalid")
void shouldThrowWhenNameIsInvalid() {
    assertThatThrownBy(() ->
            new SampleEntity(
                    SampleFixture.DEFAULT_ID,
                    ""
            )
    ).isInstanceOf(DomainException.class);
}
```

---

## ❌ Incorreto — literais crus em vez das constantes `DEFAULT_*`

```java
@Test
@DisplayName("Create sample")
void shouldCreateSampleWhenDataIsValid() {

    var sample = new SampleEntity(
            "1",
            "Sample"
    );
}
```

---

## ❌ Incorreto — sem asserções

```java
@Test
@DisplayName("Create sample")
void shouldCreateSample() {
    SampleFixture.aSample();
}
```

---

## ❌ Incorreto

```java
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
- Utilizar nomes descritivos para os métodos de teste.
- Manter os testes independentes entre si.

---

## Resumo das Convenções

Todo teste de Entidade deve respeitar os seguintes princípios:

- Cada entidade possui sua própria classe de teste.
- Apenas a camada Domain deve ser testada.
- Usar a fixture `aSample()` nos testes de comportamento, invariantes e estado; nos testes de criação, instanciar direto usando as constantes `DEFAULT_*`.
- Utilizar AssertJ para todas as validações.
- Todo método deve possuir `@DisplayName`.
- Um cenário por teste.
- Todos os comportamentos públicos devem ser cobertos.
- Os testes devem ser rápidos, determinísticos e independentes.
