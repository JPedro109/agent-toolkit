# Convenções da Camada Presentation

A camada **Presentation** é responsável exclusivamente pela comunicação entre a aplicação e consumidores externos.

Ela concentra tudo que representa uma forma de acesso à aplicação, **independentemente do protocolo ou mecanismo** (HTTP, Scheduler, mensageria, CLI, etc.). Cada tipo de acesso recebe seu estímulo de entrada, converte os dados para os DTOs da camada Application, executa os casos de uso e transforma o resultado em uma resposta apropriada ao seu meio.

A camada Presentation **não implementa regras de negócio** e **não acessa diretamente componentes da camada External**.

Este documento define os princípios **transversais** da camada — válidos para qualquer adaptador de entrada. As regras específicas de cada tipo de acesso ficam em documentos próprios por extensão (ver "Documentos por tipo de acesso").

---

# Princípios transversais

Todo adaptador de entrada desta camada, seja qual for o protocolo, deve respeitar:

- **Sem regra de negócio.** Nenhuma invariante ou decisão de domínio vive aqui. O adaptador apenas traduz entre o mundo externo e os casos de uso.
- **Dependência exclusiva da camada Application.** O adaptador depende apenas de casos de uso (classes concretas da camada Application). Nunca acessa repositórios, adaptadores, DAOs, modelos de persistência ou qualquer componente da External.
- **Orquestração em quatro passos.** Receber o estímulo de entrada → converter para o Input DTO do caso de uso → executar o caso de uso → traduzir o resultado para a resposta apropriada ao meio. A conversão de entrada e de saída é apenas cópia de campos, sem regra.
- **Tratamento de erro na borda.** Falhas são tratadas de forma centralizada no próprio limite da camada, traduzidas para o formato de erro adequado ao protocolo. O adaptador não embute lógica de recuperação de negócio.
- **Observabilidade.** O adaptador registra o início e o fim da operação (com a entrada e a saída, quando existirem), mantendo o logging concentrado nesta camada. Nos logs, usar string interpolation do Kotlin (`"$variavel"`).

---

# Estrutura

A organização da camada Presentation deve refletir as formas de acesso expostas pela aplicação. Cada tipo de acesso ocupa seu próprio subpacote.

```text
presentation/
└── controller/     # acesso HTTP (ver documento de controller)
    ├── SampleController.kt
    ├── documentation/
    ├── payload/
    └── common/
```

> Novos tipos de acesso (ex.: `scheduler/`) nascem como subpacotes irmãos de `controller/`, cada um regido pelo seu documento de extensão.

> Configuração técnica de bootstrap transversal (segurança, JWT, Swagger, filtros de requisição, tracing) **não** reside em `presentation/`: vive no pacote transversal `configuration/`, fora de `core/`, conforme o documento de arquitetura.

---

# Dependências

A camada Presentation pode depender apenas da camada Application.

Não é permitido depender diretamente de:

- Repositórios;
- Adaptadores;
- DAOs;
- Modelos de persistência;
- Componentes da External.

Toda comunicação deve ocorrer exclusivamente através dos casos de uso.

---

# Resumo das Convenções

- A camada Presentation concentra qualquer forma de acesso à aplicação, independentemente do protocolo.
- Nenhum adaptador de entrada implementa regra de negócio.
- Todo adaptador depende exclusivamente da camada Application.
- A orquestração segue quatro passos: receber → converter para Input DTO → executar o caso de uso → traduzir o resultado.
- Erros são tratados de forma centralizada na borda da camada.
- Início e fim de cada operação devem ser registrados nesta camada.
- Cada tipo de acesso é regido por seu próprio documento de extensão.
