# Convenções da Camada Application

A camada **Application** é responsável por orquestrar os casos de uso da aplicação.

Ela coordena a execução das regras de negócio da camada Domain e realiza validações ou decisões que dependam de recursos externos.

Regras que possam ser implementadas exclusivamente pelo domínio não devem ser implementadas nesta camada.

---

# Estrutura

A organização da camada Application deve refletir os casos de uso da aplicação.

A estrutura recomendada é:

```text
application/
├── port/
│   ├── persistence/
│   │   └── repository/
│   └── security/
└── usecase/
    └── sample/
        ├── dto/
        │   ├── input/
        │   └── output/
        ├── exception/
        └── CreateSampleUseCase.java
```

As seguintes regras devem ser respeitadas:

- Cada contexto deve possuir seu próprio pacote.
- Cada caso de uso é uma classe concreta localizada diretamente no pacote do contexto.
- Casos de uso não possuem interface nem sufixo `Impl`, pois representam uma operação única sem ponto de variação. Contratos (interfaces) são reservados às portas.
- DTOs devem permanecer separados entre entrada e saída.
- Portas devem permanecer organizadas por responsabilidade.

---

# Casos de Uso

Casos de uso representam operações executadas pela aplicação.

Cada caso de uso deve representar apenas uma responsabilidade.

As seguintes regras devem ser respeitadas:

- Cada caso de uso é uma classe concreta, sem interface e sem sufixo `Impl`.
- O nome da classe descreve a operação e termina em `UseCase` (ex.: `CreateSampleUseCase`).
- Como não há interface, o método `execute` não utiliza `@Override`.
- Cada caso de uso deve executar apenas um fluxo da aplicação.
- Casos de uso não devem conter regras de infraestrutura.
- Casos de uso não devem depender da camada Presentation.

> Casos de uso possuem uma única implementação e nenhum ponto de variação, por isso a interface seria pura cerimônia. A inversão de dependência é aplicada apenas às portas (repositories, security, gateways), que permanecem interfaces.

## ✔ Correto

```java
@Service
public class CreateSampleUseCase {

    public CreateSampleOutputDto execute(CreateSampleInputDto input) {
        ...
    }
}
```

## ❌ Incorreto

```java
public interface CreateSampleUseCase {
    CreateSampleOutputDto execute(CreateSampleInputDto input);
}

@Service
public class CreateSampleUseCaseImpl implements CreateSampleUseCase {
    ...
}
```

## ❌ Incorreto

```java
@Service
public class SampleService {
    public void create(...) {}
    public void update(...) {}
    public void delete(...) {}
}
```

---

# DTOs

DTOs representam exclusivamente os dados trafegados entre camadas.

Não representam regras de negócio.

As seguintes regras devem ser respeitadas:

- Devem ser imutáveis.
- Devem utilizar `record`.
- Não devem possuir validações.
- Não devem possuir regras de negócio.
- Não devem possuir dependência de frameworks.

## ✔ Correto

```java
public record CreateSampleInputDto(
        String name
) { }
```

```java
public record CreateSampleOutputDto(
        String id,
        String name
) { }
```

## ❌ Incorreto

```java
public class CreateSampleInputDto {
    @NotBlank
    private String name;
}
```

---

# Portas

Portas representam contratos entre a camada Application e componentes externos.

Toda comunicação com infraestrutura deve ocorrer através de portas.

As seguintes regras devem ser respeitadas:

- Portas devem ser interfaces.
- Portas devem representar comportamento.
- Não devem possuir implementação.
- Devem permanecer organizadas por responsabilidade.

## ✔ Correto

```java
public interface SampleRepository {
    SampleEntity save(SampleEntity entity);

    Optional<SampleEntity> findById(IdValueObject id);

    boolean existsByName(SampleNameValueObject name);
}
```

```java
public interface PasswordEncoder {
    String encode(String value);

    boolean matches(String raw, String encoded);
}
```

## ❌ Incorreto

```java
public class SampleRepository { }
```

---

# Serviços

A camada Application não deve possuir serviços genéricos.

O comportamento deve ser representado através de casos de uso.

Caso um componente seja reutilizado por múltiplos casos de uso, ele deve possuir uma responsabilidade claramente definida.

## ✔ Correto

```java
@Service
public class CreateSampleUseCase {

    public CreateSampleOutputDto execute(CreateSampleInputDto input) {
        ...
    }
}
```

## ❌ Incorreto

```java
@Service
public class CreateSampleUseCaseImpl implements CreateSampleUseCase {
    ...
}
```

## ❌ Incorreto

```java
@Service
public class SampleService {
    public void create(){}
    public void update(){}
    public void delete(){}
}
```

---

# Fluxo de Dados

A camada Application é responsável por coordenar o fluxo entre as portas e o domínio.

O fluxo recomendado é:

```text
Input DTO
      ↓
Caso de Uso
      ↓
Value Objects
      ↓
Entidade
      ↓
Porta
      ↓
Output DTO
```

As seguintes regras devem ser respeitadas:

- O caso de uso recebe um DTO de entrada.
- O caso de uso cria os Value Objects necessários.
- O caso de uso instancia a entidade.
- O caso de uso utiliza apenas portas para acessar infraestrutura.
- O retorno deve ocorrer através de um DTO de saída.

## ✔ Correto

```java
public CreateSampleOutputDto execute(
        CreateSampleInputDto input
) {
    var entity = new SampleEntity(
            UUID.randomUUID().toString(),
            input.name()
    );

    var saved = repository.save(entity);

    return new CreateSampleOutputDto(
            saved.getId().asString(),
            saved.getName().asString()
    );
}
```

---

# Conversões

A camada Application é responsável pela conversão entre DTOs e objetos do domínio.

As seguintes regras devem ser respeitadas:

- DTOs nunca devem atravessar para a camada Domain.
- Entidades nunca devem ser utilizadas diretamente pela camada Presentation.
- Conversões devem permanecer centralizadas no caso de uso ou em componentes específicos de mapeamento quando houver reutilização.

## ✔ Correto

```java
var entity = new SampleEntity(input.id(), input.name());
```

```java
return new CreateSampleOutputDto(entity.getId().asString(), entity.getName().asString());
```

## ❌ Incorreto

```java
repository.save(input);
```

```java
return entity;
```

---

# Gerenciamento Transacional

Quando um caso de uso coordena **mais de uma operação de persistência**, essas operações devem ocorrer dentro de uma única transação, garantindo atomicidade ("tudo ou nada") e uma visão consistente dos dados.

O gerenciamento transacional é controlado pela anotação `@Transactional` do Spring (`org.springframework.transaction.annotation.Transactional`), aplicada diretamente no método `execute` do caso de uso.

> **Exceção arquitetural consciente.** `@Transactional` é a única dependência de framework de persistência permitida na camada Application. A fronteira transacional é, por natureza, uma responsabilidade de orquestração do caso de uso, e abstraí-la por uma porta própria (ex.: `TransactionExecutor`/`UnitOfWork`) adicionaria indireção sem ganho prático nesta arquitetura. Essa exceção está registrada também no documento de arquitetura.

## Critério

O critério é o **número de invocações de métodos de porta de persistência dentro do `execute`** — não o número de comandos SQL gerados. Uma única invocação de porta pode gerar vários SQLs e ainda assim contar como uma só chamada.

As seguintes regras devem ser respeitadas:

- **1 invocação de porta de persistência** → sem `@Transactional`. A operação isolada já é atômica pela transação do próprio Spring Data; anotar não é erro, mas não agrega.
- **2 ou mais invocações, todas de leitura** → `@Transactional(readOnly = true)`. Garante que todas as leituras enxerguem o mesmo snapshot consistente e sinaliza a intenção de leitura.
- **2 ou mais invocações, contendo ao menos uma escrita** → `@Transactional`. É o caso crítico: sem a transação abrangente, uma falha após a primeira escrita deixaria o banco em estado parcial e irreversível.

## Por que o número de chamadas é o gatilho

Com uma única chamada não há nada a coordenar. A partir de duas, surge o risco de o conjunto falhar no meio: a primeira operação commita, a segunda falha, e o banco fica inconsistente. `@Transactional` transforma as N operações em um bloco indivisível, confirmado por inteiro no commit único do fim do método, ou revertido por inteiro no rollback.

## ✔ Correto

```java
@Service
public class UpdateSampleUseCase {

    @Transactional
    public SampleOutputDto execute(UpdateSampleInputDto input) {
        var sample = repository.findById(IdValueObject.of(input.id()).getValueOrThrow())
                .orElseThrow(SampleNotFoundException::new);

        sample.updateName(input.name());

        var saved = repository.save(sample);

        return new SampleOutputDto(saved.getId().asString(), saved.getName().asString());
    }
}
```

```java
@Service
public class ListSamplesUseCase {

    @Transactional(readOnly = true)
    public List<SampleOutputDto> execute(ListSamplesInputDto input) {
        ...
    }
}
```

## ❌ Incorreto

```java
@Service
public class CreateSampleUseCase {

    // Duas escritas sem transação abrangente: se a segunda falhar,
    // a primeira já foi confirmada e o estado fica parcial.
    public SampleOutputDto execute(CreateSampleInputDto input) {
        repository.save(firstEntity);
        auditRepository.save(secondEntity);
        ...
    }
}
```

---

# Dependências

A camada Application pode depender apenas da camada Domain e de contratos definidos pela própria Application.

Não é permitido depender de:

- Controllers;
- Requests;
- Responses;
- Models de persistência;
- Repositórios concretos;
- Frameworks de persistência, com a **única exceção** da anotação `@Transactional` do Spring para o gerenciamento transacional dos casos de uso (ver seção "Gerenciamento Transacional");
- Componentes da camada External.

---

# Resumo das Convenções

Toda implementação da camada Application deve respeitar os seguintes princípios:

- Cada caso de uso representa uma única operação da aplicação.
- Cada caso de uso é uma classe concreta, sem interface e sem sufixo `Impl`.
- Contratos (interfaces) são reservados às portas, nunca aos casos de uso.
- DTOs devem utilizar `record`.
- DTOs não possuem regras de negócio.
- Toda comunicação externa deve ocorrer através de portas.
- A camada Application coordena o fluxo da aplicação, executa validações que dependam de recursos externos e orquestra a execução das regras de negócio da camada Domain.
- Entidades pertencem ao Domain.
- Componentes da External nunca devem ser acessados diretamente.
- O retorno dos casos de uso deve ocorrer através de DTOs de saída.
- Casos de uso com 2 ou mais invocações de portas de persistência devem ser transacionais: `@Transactional(readOnly = true)` quando forem todas de leitura e `@Transactional` quando houver ao menos uma escrita. A anotação `@Transactional` do Spring é a única dependência de framework de persistência permitida nesta camada.
