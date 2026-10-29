# Arquitetura do Projeto

## Princípios arquiteturais

Toda implementação deve respeitar os seguintes princípios:

* O domínio representa o núcleo da aplicação e contém exclusivamente regras de negócio.
* Regras de negócio nunca dependem de frameworks, bibliotecas ou tecnologias externas.
* Casos de uso representam operações completas da aplicação e coordenam a execução das regras de negócio.
* Infraestrutura implementa apenas detalhes técnicos necessários para suportar a aplicação.
* A camada de apresentação apenas recebe requisições, delega sua execução e devolve respostas.
* A comunicação com a infraestrutura ocorre através de contratos (portas) bem definidos.
* Dependências de infraestrutura nunca devem ser representadas por implementações concretas quando uma porta puder representar a dependência.
* Contratos existem para inverter dependências entre camadas (portas e adaptadores), não para componentes de implementação única como os casos de uso.
* A única exceção à proibição de dependência de frameworks de persistência na camada Application é a anotação `@Transactional` do Spring, aplicada aos casos de uso que coordenam mais de uma operação de persistência. A fronteira transacional é uma responsabilidade de orquestração do caso de uso, e abstraí-la por uma porta dedicada adicionaria indireção sem ganho prático. O critério e as regras estão definidos no documento de convenções da camada Application.

---

## Estrutura de Pastas

A organização física do projeto deve refletir a arquitetura definida aqui. Cada camada deve possuir um espaço próprio, evitando a mistura de responsabilidades e mantendo uma separação clara entre os componentes da aplicação.

A estrutura do projeto é:

```text
core/
├── domain/
├── application/
├── external/
└── presentation/
```

As quatro camadas de negócio residem sob `core/`. Além delas, o projeto possui um pacote transversal `configuration/`, que concentra a configuração técnica da aplicação (segurança, filtros, documentação, tracing) e não pertence a nenhuma das camadas de negócio:

```text
configuration/
├── filter/      # filtros de requisição (ex.: autenticação JWT, contexto de tracing)
├── security/    # configuração do Spring Security e propriedades tipadas
├── swagger/     # configuração do SpringDoc/OpenAPI
└── tracing/     # configuração de observabilidade/baggage
```

O pacote `configuration/` é a fronteira técnica de bootstrap do framework. Componentes transversais que precisam ser registrados no contexto do Spring e que não implementam regra de negócio (filtros, beans de configuração) residem aqui, e não dentro das camadas de `core/`.

---

## Fluxo de dependências

As dependências sempre apontam para o centro da arquitetura.

A direção permitida das dependências é:

```text
Presentation
      ↓
Application
      ↓
Domain

External
      ↓
Application
      ↓
Domain
```

Portanto:

* Domain não depende de nenhuma outra camada.
* Application depende apenas de Domain.
* External depende de Application e Domain.
* Presentation depende de Application.

Nenhuma camada interna conhece implementações localizadas em camadas externas.

---

## Fluxo de dados

O fluxo de dados entre as camadas deve ser unidirecional.

```text
Request
    ↓
Input DTO
    ↓
Domain
    ↓
Output DTO
    ↓
Response
```

Cada camada trabalha exclusivamente com seus próprios objetos.

Objetos pertencentes a uma camada nunca devem atravessar diretamente outra camada sem conversão explícita.

As conversões entre objetos devem ocorrer exatamente na fronteira entre duas camadas.

---

## Responsabilidades das camadas

Cada camada possui responsabilidades exclusivas.

### Domain

Responsável por:

* regras de negócio;
* entidades;
* value objects;
* validações de domínio;
* invariantes;
* comportamentos do domínio.

Não é responsável por:

* persistência;
* autenticação;
* protocolos de comunicação;
* frameworks;
* acesso ao banco de dados;
* detalhes de infraestrutura.

---

### Application

Responsável por:

* casos de uso;
* orquestração da aplicação;
* definição de contratos;
* coordenação entre domínio e infraestrutura.

Não é responsável por:

* regras de infraestrutura;
* detalhes de frameworks;
* protocolos externos.

---

### External

Responsável por:

* persistência;
* autenticação;
* integrações externas;
* mensageria;
* armazenamento;
* cache;
* implementação das portas definidas pela aplicação.

Não é responsável por:

* regras de negócio.

---

### Presentation

Responsável por:

* receber entradas externas;
* validar dados de entrada;
* converter dados para DTOs da aplicação;
* chamar casos de uso;
* converter respostas para o formato externo adequado.

Não é responsável por:

* implementar regras de negócio;
* acessar infraestrutura diretamente;
* realizar persistência.

---

## Regras gerais de desacoplamento

Todas as implementações devem seguir as seguintes regras:

* Cada classe deve possuir uma única responsabilidade.
* Dependências que cruzam a fronteira com a infraestrutura devem depender de abstrações (portas).
* Nenhuma camada pode acessar diretamente detalhes internos de outra camada.
* Toda integração com tecnologias externas deve ser encapsulada em adaptadores.
* Toda conversão entre modelos deve ser explícita.
* Nenhuma regra de negócio deve depender de detalhes técnicos.
* Nenhuma implementação de infraestrutura deve ser utilizada diretamente quando existir uma porta equivalente. Componentes de implementação única, como casos de uso, são utilizados diretamente por não possuírem ponto de variação.
* Objetos pertencentes a uma camada não devem ser reutilizados por outra camada sem uma conversão explícita.
* Toda dependência deve possuir a menor abrangência possível.
* O acoplamento entre módulos deve ser minimizado e a coesão interna maximizada.
