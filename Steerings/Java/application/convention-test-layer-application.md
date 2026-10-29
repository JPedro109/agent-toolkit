# Testes Unitários de Casos de Uso

Os testes de **Casos de Uso** têm como objetivo garantir que o fluxo da aplicação seja executado corretamente.

Eles devem validar exclusivamente o comportamento da camada **Application**, assegurando a correta orquestração entre o domínio e as portas da aplicação.

Casos de uso devem ser testados isoladamente, sem carregar o contexto do Spring e sem acessar componentes reais de infraestrutura.

---

## Estrutura

Os testes devem permanecer organizados conforme o caso de uso.

Exemplo:

```text
src/test/java/
└── core/
    └── application/
        └── usecase/
            └── sample/
                └── CreateSampleUseCaseTest.java
```

---

## Convenções

As seguintes regras devem ser respeitadas:

- Cada caso de uso deve possuir sua própria classe de teste.
- Dependências devem ser simuladas utilizando Mockito.
- O contexto do Spring não deve ser iniciado.
- Apenas a classe do caso de uso deve ser instanciada.
- Todos os fluxos de sucesso e falha devem ser testados.
- O comportamento das portas deve ser validado através de verificações (`verify`).
- Fixtures devem ser utilizadas sempre que possível.
- Cada teste deve validar apenas um cenário.

---

## Dependências

Casos de uso devem ser testados utilizando apenas:

- JUnit;
- Mockito;
- Fixtures;
- Objetos do domínio.

Não devem utilizar:

- Spring Boot Test;
- Banco de dados;
- Testcontainers;
- Componentes reais da External.

---

## Cenários obrigatórios

Todo caso de uso deve possuir, no mínimo, testes para:

- Execução com sucesso.
- Validação de regras do domínio.
- Validação de regras da aplicação.
- Exceções lançadas pelo domínio.
- Exceções lançadas por dependências externas.
- Interação correta com as portas.
- Retorno esperado.

---

## ✔ Correto

```java
@ExtendWith(MockitoExtension.class)
class CreateSampleUseCaseTest {

    @Mock
    private SampleRepository repository;

    @InjectMocks
    private CreateSampleUseCase useCase;

    @Test
    @DisplayName("Should create sample")
    void shouldCreateSample() {
        var sample = SampleFixture.aSample();
        var name = sample.getName();
        var input = new CreateSampleInputDto(name.asString());

        when(repository.save(any())).thenAnswer(invocation -> invocation.getArgument(0));

        var output = useCase.execute(input);

        assertNotNull(output);
        assertEquals(name.asString(), output.name());
        verify(repository).save(any());
    }
}
```

---

## ✔ Correto

```java
@Test
@DisplayName("Should throw exception when sample already exists")
void shouldThrowExceptionWhenSampleAlreadyExists() {
    var sample = SampleFixture.aSample();
    var name = sample.getName();
    var input = new CreateSampleInputDto(name.asString());

    when(repository.existsByName(any())).thenReturn(true);

    assertThrows(
            SampleAlreadyExistsException.class,
            () -> useCase.execute(input)
    );
    verify(repository, never()).save(any());
}
```

---

## ✔ Correto

```java
@Test
void shouldPersistEntity() {
    var sample = SampleFixture.aSample();
    var name = sample.getName();
    var input = new CreateSampleInputDto(name.asString());

    useCase.execute(input);

    verify(repository).save(any(SampleEntity.class));
}
```

---

## ❌ Incorreto

```java
@SpringBootTest
class CreateSampleUseCaseTest { }
```

---

## ❌ Incorreto

```java
@Test
@DisplayName("Should create sample")
void shouldCreateSample() {
    var repository = new SampleRepositoryAdapter(...);
}
```

---

## ❌ Incorreto

```java
@Test
@DisplayName("Should create sample")
void shouldCreateSample() {
    useCase.execute(input);
}
```

---

## Boas práticas

Sempre que possível:

- Utilizar Fixtures para criação dos objetos.
- Validar o retorno do caso de uso.
- Validar as interações com as dependências.
- Utilizar `verifyNoMoreInteractions()` quando aplicável.
- Um cenário por teste.
- Evitar múltiplos asserts para comportamentos distintos.
- Nomear os testes de forma descritiva.

---

## Resumo das Convenções

Todo teste de Caso de Uso deve respeitar os seguintes princípios:

- Cada caso de uso possui sua própria classe de teste.
- Apenas a camada Application deve ser testada.
- Dependências devem ser simuladas utilizando Mockito.
- O contexto do Spring não deve ser iniciado.
- Fixtures devem ser reutilizadas sempre que possível.
- Todo fluxo de sucesso e falha deve ser validado.
- Toda interação com as portas deve ser verificada.
- Cada teste deve validar um único cenário.
- Testes devem ser rápidos, determinísticos e independentes.
