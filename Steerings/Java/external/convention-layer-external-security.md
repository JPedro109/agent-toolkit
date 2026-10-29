---
inclusion: manual
---

# Convenções de Segurança (Java)

Esta é a versão **Java** das convenções de segurança. Os princípios arquiteturais são idênticos aos da versão Kotlin (`convention-security.md`); o que muda são as formas idiomáticas da linguagem: `record` para DTOs e propriedades, enums com construtor, `private static final` para constantes e logger, injeção por construtor com campos `final` e tratamento de exceção com `try/catch`.

A **segurança** é um detalhe de infraestrutura responsável por autenticar requisições, autorizar o acesso aos recursos e emitir/validar os tokens que sustentam a sessão sem estado (stateless) da aplicação.

Diferente de outras preocupações técnicas, a segurança se distribui por três locais bem definidos, cada um com uma responsabilidade distinta. Nenhuma regra de negócio deve conhecer detalhes de segurança: as camadas Domain e Application dependem apenas de **contratos** (portas), nunca de JWT, Spring Security ou de qualquer biblioteca de criptografia.

Esta convenção está organizada em dois blocos: primeiro a **autenticação** (quem é o usuário e como o token é emitido e validado) e, em seguida, a **autorização** (o que o usuário autenticado pode acessar).

Os componentes cobertos por esta convenção são:

- `SecurityConfig` — configuração do Spring Security (filter chain, autorização, sessão).
- `SecurityConfigProperties` — propriedades tipadas de segurança (JWT).
- `Token` — porta que define o contrato de emissão e validação de tokens.
- `TokenType` — enum de domínio da aplicação que tipifica os tokens (`access` / `refresh`).
- `TokenAdapter` — adaptador que implementa a porta `Token` com a biblioteca de JWT.
- `JwtAuthenticationFilter` — filtro que autentica a requisição a partir do token.
- Exceções de segurança — `InvalidTokenException`.
- Autorização de endpoints e por método — regras de acesso do Spring (`permitAll`, `authenticated`, `hasRole`, `hasAuthority`, `@PreAuthorize`).

---

# Convenções Idiomáticas do Java

Estas são as diferenças de estilo aplicadas em toda esta convenção, em relação à versão Kotlin:

- **DTOs e propriedades** usam `record`, obtendo imutabilidade, construtor canônico, `equals`/`hashCode`/`toString` automáticos.
- **Constantes e logger** são `private static final`; o logger é `private static final Logger` no topo da classe.
- **Injeção de dependência** é por construtor, com campos `private final`. Não usar injeção por campo (`@Autowired` em atributo).
- **Tratamento de exceção** usa `try/catch` explícito; a tradução de exceções da biblioteca ocorre no `catch`.
- **Inicialização preguiçosa** de recursos custosos (ex.: a chave de assinatura) é feita no construtor, uma única vez, e guardada em campo `final`.
- **Nomes de siglas** seguem o estilo Java usual, mas mantemos o padrão do projeto para o sufixo `Dto` (ex.: `DecodeTokenDto`).

---

# Onde Cada Componente Reside

A segurança atravessa a fronteira entre o bootstrap técnico do framework e as camadas de negócio. Por isso, seus componentes residem em três pacotes distintos, e essa separação **não** é acidental:

```text
configuration/security/          # bootstrap técnico do Spring Security (fora de core/)
├── SecurityConfig.java
└── SecurityConfigProperties.java

core/application/port/security/  # contratos (portas) conhecidos pelo negócio
├── Token.java
├── TokenType.java
├── dto/
│   └── DecodeTokenDto.java
└── exception/
    └── InvalidTokenException.java

core/external/security/          # adaptadores que implementam as portas
├── TokenAdapter.java
└── PasswordEncoderAdapter.java
```

As seguintes regras devem ser respeitadas:

- A configuração técnica de bootstrap (`SecurityConfig`, `SecurityConfigProperties`) reside no pacote transversal `configuration/security/`, fora de `core/`, conforme o documento de arquitetura.
- Os contratos de segurança (`Token`, `TokenType`, `PasswordEncoder`) residem em `application/port/security/`, pois representam comportamento conhecido pelo negócio.
- As implementações concretas dos contratos (`TokenAdapter`, `PasswordEncoderAdapter`) residem em `external/security/`, por integrarem bibliotecas externas.
- Os filtros de requisição (ex.: `JwtAuthenticationFilter`) residem em `configuration/filter/`, pois são componentes transversais de bootstrap registrados no contexto do Spring.

---

# Fluxo de Dependências

A segurança respeita a mesma inversão de dependência de todo o projeto: o negócio depende de contratos, e a infraestrutura os implementa.

```text
Presentation / Filter
            ↓
    Token (porta)        ← contrato em application/port/security
            ↑
      TokenAdapter       ← implementação em external/security
            ↓
   Biblioteca JWT / Spring Security
```

As seguintes regras devem ser respeitadas:

- As camadas Domain e Application nunca dependem de JWT, Spring Security ou de qualquer biblioteca de segurança.
- Todo acesso à segurança a partir do negócio ocorre exclusivamente através das portas (`Token`, `PasswordEncoder`).
- O `SecurityConfig` e o `TokenAdapter` podem depender de frameworks, pois são infraestrutura.

---

# Autenticação

Esta seção reúne todos os componentes responsáveis por autenticar a requisição: a configuração do Spring Security, suas propriedades tipadas, o contrato de token, o tipo de token, o adaptador que emite/valida o JWT, o filtro que autentica a requisição e as exceções de autenticação.

---

## SecurityConfig

O `SecurityConfig` é a classe de bootstrap do Spring Security. É responsável por declarar a filter chain, a política de sessão, as regras de autorização dos endpoints e o encoder de senha exposto como bean.

As seguintes regras devem ser respeitadas:

- Deve ser anotada com `@Configuration` e `@EnableWebSecurity`.
- A sessão deve ser sempre `STATELESS` — a aplicação não mantém sessão em servidor, apenas valida o token a cada requisição.
- Os endpoints públicos devem ser declarados explicitamente com `permitAll()`; todo o restante deve exigir autenticação via `anyRequest().authenticated()`.
- O `JwtAuthenticationFilter` deve ser registrado antes do `UsernamePasswordAuthenticationFilter`.
- O `SecurityConfig` não deve conter regras de negócio nem lógica de emissão/validação de token — essa responsabilidade pertence ao `TokenAdapter`.
- Falhas de autenticação devem resultar em `401 Unauthorized` através de um `authenticationEntryPoint`.

### ✔ Correto

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public BCryptPasswordEncoder bCryptPasswordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Bean
    public SecurityFilterChain securityFilterChain(
            HttpSecurity http,
            JwtAuthenticationFilter jwtFilter) throws Exception {
        return http
                .csrf(csrf -> csrf.disable())
                .sessionManagement(session ->
                        session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
                .authorizeHttpRequests(auth -> auth
                        .requestMatchers(HttpMethod.POST, "/api/v1/auth/login").permitAll()
                        .anyRequest().authenticated())
                .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class)
                .exceptionHandling(ex -> ex.authenticationEntryPoint(
                        (request, response, authException) ->
                                response.sendError(HttpServletResponse.SC_UNAUTHORIZED)))
                .build();
    }
}
```

### ❌ Incorreto — regra de autorização implícita ou endpoint aberto por omissão

```java
.authorizeHttpRequests(auth -> auth
        .requestMatchers("/api/v1/auth/login").permitAll())
        // Faltando anyRequest().authenticated(): o restante fica aberto por omissão.
```

---

## SecurityConfigProperties

As propriedades de segurança devem ser externalizadas e tipadas, nunca codificadas na aplicação ou acessadas via `@Value`.

As seguintes regras devem ser respeitadas:

- Deve utilizar `@ConfigurationProperties` com o prefixo `security`.
- Deve possuir o sufixo `ConfigProperties`, conforme a convenção de código.
- Deve ser um `record`, obtendo imutabilidade e binding por construtor (`@ConfigurationProperties` com construtor canônico).
- Subgrupos aninhados no YAML (ex.: `jwt`) devem ser representados por um `record` aninhado.
- Deve residir em `configuration/security/`.
- Todos os valores devem ser controlados por variáveis de ambiente, com valores padrão adequados ao ambiente local.
- O segredo JWT deve possuir no mínimo 32 caracteres e jamais deve ser versionado com um valor real — o default existe apenas para o ambiente local e deve ser substituído em produção.
- Deve ser registrada centralmente via `@EnableConfigurationProperties` na classe principal da aplicação.

### ✔ Correto

```java
@ConfigurationProperties(prefix = "security")
public record SecurityConfigProperties(Jwt jwt) {

    public record Jwt(
            String secret,
            long accessTokenExpirationMs,
            long refreshTokenExpirationMs) {
    }
}
```

```yaml
security:
  jwt:
    secret: ${JWT_SECRET:change-me-in-production-must-be-at-least-32-chars}
    access-token-expiration-ms: ${JWT_ACCESS_EXPIRATION_MS:900000}
    refresh-token-expiration-ms: ${JWT_REFRESH_EXPIRATION_MS:604800000}
```

### ❌ Incorreto — acesso direto via `@Value`

```java
@Service
public class TokenAdapter {

    @Value("${security.jwt.secret}")
    private String secret;
}
```

---

## Token (Porta)

A porta `Token` define o contrato de emissão e validação de tokens. É o único ponto pelo qual o negócio interage com a segurança baseada em token.

As seguintes regras devem ser respeitadas:

- Deve ser uma interface, sem implementação, em `application/port/security/`.
- Deve expressar comportamento em termos do negócio (gerar/validar access e refresh), nunca expor tipos da biblioteca de JWT.
- A validação deve retornar um DTO próprio da porta (`DecodeTokenDto`), nunca um `Claims` ou tipo equivalente da biblioteca.
- Deve trabalhar exclusivamente com tipos primitivos/strings e DTOs da própria porta.
- O DTO da porta deve ser um `record`.

### ✔ Correto

```java
public interface Token {

    String generateAccessToken(String sub);

    String generateRefreshToken(String sub);

    DecodeTokenDto validateAccessToken(String token);

    DecodeTokenDto validateRefreshToken(String token);
}
```

```java
public record DecodeTokenDto(String sub) {
}
```

### ❌ Incorreto — vazamento de tipo da biblioteca na porta

```java
public interface Token {

    Claims validateAccessToken(String token); // tipo do io.jsonwebtoken vazando para o negócio
}
```

---

## TokenType

O `TokenType` é um enum da aplicação que tipifica os tokens emitidos, diferenciando o token de acesso do token de atualização. Reside em `application/port/security/`, junto do contrato `Token`, pois é parte do vocabulário de segurança conhecido pelo negócio.

As seguintes regras devem ser respeitadas:

- Deve ser um `enum` cujo valor textual (`value`) é o que de fato trafega no token, nunca o nome da constante Java (`name()`).
- O `value` é um campo `final` exposto por getter, definido no construtor do enum.
- A conversão de string para enum deve ser centralizada em um método estático (`fromValue`).
- Um valor desconhecido ou nulo na conversão deve resultar em `InvalidTokenException`, nunca em `null` silencioso ou exceção da biblioteca.
- Não deve conhecer detalhes da biblioteca de JWT — apenas representa o tipo do token.

### ✔ Correto

```java
public enum TokenType {

    ACCESS("access"),
    REFRESH("refresh");

    private final String value;

    TokenType(String value) {
        this.value = value;
    }

    public String getValue() {
        return value;
    }

    public static TokenType fromValue(String value) {
        return Arrays.stream(values())
                .filter(type -> type.value.equals(value))
                .findFirst()
                .orElseThrow(InvalidTokenException::new);
    }
}
```

### ❌ Incorreto — uso do nome da constante como valor do token e conversão insegura

```java
public enum TokenType {

    ACCESS,
    REFRESH;

    // Depende de name() e lança a exceção crua da enum (IllegalArgumentException).
    public static TokenType fromValue(String value) {
        return TokenType.valueOf(value.toUpperCase());
    }
}
```

---

## TokenAdapter

O `TokenAdapter` é o adaptador que implementa a porta `Token` utilizando a biblioteca de JWT. É o único componente autorizado a conhecer a biblioteca de tokens.

As seguintes regras devem ser respeitadas:

- Deve residir em `external/security/`, implementar a porta `Token` e ser registrado como `@Component`.
- Deve receber a configuração via `SecurityConfigProperties` injetada no construtor, nunca via `@Value`.
- O valor textual do `TokenType` (`type.getValue()`) é o que deve ser gravado e lido do claim do token; o adaptador converte de volta via `TokenType.fromValue`.
- Toda exceção proveniente da biblioteca de JWT deve ser capturada em `try/catch` e traduzida para `InvalidTokenException` antes de atravessar a fronteira da camada External. A camada de negócio nunca deve conhecer exceções da biblioteca.
- A exceção original da biblioteca deve ser registrada em log (nível `WARN`) no ponto da tradução, nunca em outra camada. Nunca registrar o token, o segredo ou qualquer dado sensível.
- A chave de assinatura e demais recursos custosos devem ser derivados uma única vez, no construtor, e guardados em campo `final`, nunca recriados a cada chamada.
- Nomes de claim e demais literais devem ser constantes `private static final`, conforme a convenção de código.

### ✔ Correto

```java
@Component
public class TokenAdapter implements Token {

    private static final Logger logger = LoggerFactory.getLogger(TokenAdapter.class);

    private static final String CLAIM_TOKEN_TYPE = "token_type";

    private final SecurityConfigProperties properties;
    private final SecretKey signingKey;

    public TokenAdapter(SecurityConfigProperties properties) {
        this.properties = properties;
        this.signingKey = Keys.hmacShaKeyFor(properties.jwt().secret().getBytes(StandardCharsets.UTF_8));
    }

    @Override
    public String generateAccessToken(String sub) {
        return buildToken(sub, TokenType.ACCESS, properties.jwt().accessTokenExpirationMs());
    }

    @Override
    public DecodeTokenDto validateAccessToken(String token) {
        return validateToken(token, TokenType.ACCESS);
    }

    private DecodeTokenDto validateToken(String token, TokenType expectedType) {
        try {
            Claims claims = parseClaims(token);

            TokenType tokenType = TokenType.fromValue(claims.get(CLAIM_TOKEN_TYPE, String.class));

            if (tokenType != expectedType) {
                throw new InvalidTokenException();
            }

            return new DecodeTokenDto(claims.getSubject());
        } catch (InvalidTokenException ex) {
            throw ex;
        } catch (Exception ex) {
            logger.warn("Invalid JWT token: {}", ex.getMessage());

            throw new InvalidTokenException();
        }
    }
}
```

> Observação: o `catch (InvalidTokenException ex)` precede o `catch (Exception ex)` para que a exceção de tipo incorreto — já da aplicação — propague sem ser logada novamente como falha da biblioteca.

### ❌ Incorreto — exceção da biblioteca vazando para o negócio

```java
@Override
public DecodeTokenDto validateAccessToken(String token) {
    // ExpiredJwtException / SignatureException escapam para a camada Application.
    Claims claims = Jwts.parser().verifyWith(signingKey).build().parseSignedClaims(token).getPayload();
    return new DecodeTokenDto(claims.getSubject());
}
```

### ❌ Incorreto — segredo recriado a cada chamada e dado sensível em log

```java
@Override
public DecodeTokenDto validateAccessToken(String token) {
    // Chave recriada a cada validação, em vez de derivada uma única vez no construtor.
    SecretKey key = Keys.hmacShaKeyFor(properties.jwt().secret().getBytes(StandardCharsets.UTF_8));
    logger.info("Validating token {}", token); // token sensível em log
    ...
}
```

---

## JwtAuthenticationFilter

O `JwtAuthenticationFilter` é o ponto de entrada da autenticação por token. A cada requisição, ele extrai o token do header `Authorization`, valida-o através da porta `Token` e, em caso de sucesso, popula o `SecurityContextHolder` com o usuário autenticado. É ele quem transforma um token válido em uma autenticação que o Spring Security reconhece.

O filtro reside em `configuration/filter/`, pois é um componente transversal de bootstrap registrado no contexto do Spring, e não uma camada de negócio. Ele é registrado antes do `UsernamePasswordAuthenticationFilter` no `SecurityConfig`.

As seguintes regras devem ser respeitadas:

- Deve estender `OncePerRequestFilter`, garantindo uma única execução por requisição.
- Deve ser registrado como `@Component` e depender exclusivamente da porta `Token`, nunca da biblioteca de JWT diretamente.
- Deve ler o token do header `Authorization` apenas quando o prefixo `Bearer ` estiver presente; o comprimento do prefixo deve ser uma constante `private static final`.
- A validação do token deve ser envolvida em `try/catch` — um token inválido **não** deve lançar exceção nem bloquear a cadeia. O filtro apenas deixa de autenticar e segue, delegando a resposta `401` ao `authenticationEntryPoint` do `SecurityConfig`.
- Em caso de token válido, deve construir um `UsernamePasswordAuthenticationToken` com o `sub` como principal e registrá-lo no `SecurityContextHolder`.
- O terceiro argumento do `UsernamePasswordAuthenticationToken` são as authorities do usuário. Enquanto a autorização usar apenas `authenticated()`, `List.of()` basta; ao adotar `hasRole`/`hasAuthority`, essa lista **precisa** ser populada (ver [Pré-requisito: popular as authorities na autenticação](#pré-requisito-popular-as-authorities-na-autenticação)).
- Nunca deve registrar o token ou qualquer dado sensível em log.
- Não deve conter regra de negócio nem decidir autorização — apenas estabelece quem é o usuário autenticado.

### ✔ Correto

```java
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private static final String BEARER_PREFIX = "Bearer ";

    private final Token token;

    public JwtAuthenticationFilter(Token token) {
        this.token = token;
    }

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain) throws ServletException, IOException {

        String authHeader = request.getHeader("Authorization");

        if (authHeader != null && authHeader.startsWith(BEARER_PREFIX)) {
            String jwt = authHeader.substring(BEARER_PREFIX.length());

            try {
                DecodeTokenDto decoded = token.validateAccessToken(jwt);

                var authentication = new UsernamePasswordAuthenticationToken(
                        decoded.sub(),
                        null,
                        List.of());

                authentication.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));

                SecurityContextHolder.getContext().setAuthentication(authentication);
            } catch (InvalidTokenException ex) {
                // Token inválido: segue sem autenticar; o 401 fica a cargo do authenticationEntryPoint.
            }
        }

        filterChain.doFilter(request, response);
    }
}
```

### ❌ Incorreto — token inválido bloqueando a cadeia e dependência direta da biblioteca

```java
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtParser jwtParser; // dependência direta da lib, em vez da porta Token

    public JwtAuthenticationFilter(JwtParser jwtParser) {
        this.jwtParser = jwtParser;
    }

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain) throws ServletException, IOException {

        String jwt = request.getHeader("Authorization").substring(7);

        // Exceção da lib escapa e interrompe a cadeia: o 401 deixa de ser tratado
        // pelo authenticationEntryPoint e vira um 500.
        Claims claims = jwtParser.parseSignedClaims(jwt).getPayload();

        SecurityContextHolder.getContext().setAuthentication(
                new UsernamePasswordAuthenticationToken(claims.getSubject(), null, List.of()));

        filterChain.doFilter(request, response);
    }
}
```

---

## Exceções

Toda falha de segurança relevante para o negócio deve ser representada por uma exceção própria da aplicação, nunca por uma exceção da biblioteca de JWT ou do Spring Security.

As seguintes regras devem ser respeitadas:

- A `InvalidTokenException` reside em `application/port/security/exception/`, junto do contrato que a lança.
- Estende `RuntimeException` diretamente e carrega uma mensagem fixa, conforme a convenção da camada Application.
- É a única exceção que o negócio conhece a respeito de tokens — expiração, assinatura inválida, tipo incorreto e formato inválido convergem todos para ela.
- O registro em log dessa exceção segue a convenção de código: a tradução loga no ponto da tradução (`TokenAdapter`), e demais logs de exceção ficam a cargo do Global Exception Handler.

### ✔ Correto

```java
public class InvalidTokenException extends RuntimeException {

    public InvalidTokenException() {
        super("Invalid or expired token");
    }
}
```

---

# Autorização

Esta seção reúne as regras de acesso do Spring Security: a autorização por endpoint, declarada no `authorizeHttpRequests`, e a autorização por método, via `@PreAuthorize`. Diferente da autenticação, a autorização decide **o que** o usuário já autenticado pode acessar.

---

## Autorização de Endpoints

A autorização de endpoints é declarada dentro do `authorizeHttpRequests` do `SecurityConfig` e decide **se** uma requisição pode alcançar um endpoint, com base em autenticação, papéis (`roles`) ou autoridades (`authorities`). É uma barreira de transporte, aplicada antes do controller, e nunca deve conter regra de negócio.

As regras são avaliadas **na ordem de declaração**: a primeira que casar com a requisição vence. Por isso, regras mais específicas vêm antes das mais genéricas, e `anyRequest()` é sempre a última, atuando como fecho (deny-by-default).

### Matchers de Autorização

Cada `requestMatchers(...)` (ou `anyRequest()`) é encerrado por um matcher que define a condição de acesso. Os matchers disponíveis são:

| Matcher | Assinatura | Concede acesso quando |
|---|---|---|
| `permitAll()` | `permitAll()` | Sempre — endpoint público, sem exigir autenticação. |
| `denyAll()` | `denyAll()` | Nunca — bloqueia qualquer acesso, útil para desativar uma rota. |
| `authenticated()` | `authenticated()` | O usuário está autenticado, independentemente de papel ou autoridade. |
| `hasRole(...)` | `hasRole(String role)` | O usuário possui **o** papel informado. O prefixo `ROLE_` é adicionado automaticamente. |
| `hasAnyRole(...)` | `hasAnyRole(String... roles)` | O usuário possui **ao menos um** dos papéis informados. Prefixo `ROLE_` automático. |
| `hasAuthority(...)` | `hasAuthority(String authority)` | O usuário possui **a** autoridade literal informada. **Sem** prefixo automático. |
| `hasAnyAuthority(...)` | `hasAnyAuthority(String... authorities)` | O usuário possui **ao menos uma** das autoridades literais informadas. **Sem** prefixo. |

A diferença entre `role` e `authority` é apenas o prefixo `ROLE_`: `hasRole("ADMIN")` é equivalente a `hasAuthority("ROLE_ADMIN")`. Papéis representam categorias amplas (`ADMIN`, `MANAGER`); autoridades representam permissões granulares (`task:read`, `user:delete`). Escolha um dos dois modelos e mantenha-o em todo o projeto.

### Pré-requisito: popular as authorities na autenticação

Os matchers por papel e por autoridade (`hasRole`, `hasAnyRole`, `hasAuthority`, `hasAnyAuthority`) só funcionam se o usuário autenticado **carregar** essas authorities. Elas são lidas do `Authentication` registrado pelo `JwtAuthenticationFilter` no `SecurityContextHolder`.

Hoje o filtro constrói a autenticação com uma lista vazia de authorities:

```java
var authentication = new UsernamePasswordAuthenticationToken(
        decoded.sub(),
        null,
        List.of()); // sem authorities: apenas authenticated() funciona
```

Com `List.of()`, apenas `authenticated()` e `permitAll()` casam. Qualquer regra `hasRole`/`hasAuthority` falha silenciosamente, pois o usuário não possui papel algum.

Portanto, ao adotar autorização por papel ou autoridade, é necessário:

- Propagar os papéis/autoridades do usuário no token (ex.: um claim `roles` ou `authorities`), lendo-os no `TokenAdapter` e expondo-os no `DecodeTokenDto`.
- Converter esses valores em `GrantedAuthority` e passá-los no terceiro argumento do `UsernamePasswordAuthenticationToken`, no lugar de `List.of()`.
- O valor passado para `SimpleGrantedAuthority(...)` deve ser **idêntico** ao que o matcher procura. É isso que decide se você usa `hasRole` ou `hasAuthority`.

A escolha do modelo muda apenas o prefixo:

| Modelo | Authority criada no filtro | Matcher que casa |
|---|---|---|
| Papel (role) | `new SimpleGrantedAuthority("ROLE_ADMIN")` | `hasRole("ADMIN")` — o Spring adiciona o `ROLE_` |
| Autoridade (authority) | `new SimpleGrantedAuthority("report:read")` | `hasAuthority("report:read")` — literal, sem prefixo |

#### ✔ Correto — modelo de papéis (`hasRole` / `hasAnyRole`)

Com papéis, adicione o prefixo `ROLE_` ao criar a authority, pois `hasRole("ADMIN")` procura por `ROLE_ADMIN`:

```java
List<GrantedAuthority> authorities = decoded.roles().stream()
        .map(role -> new SimpleGrantedAuthority("ROLE_" + role))
        .collect(Collectors.toList());

var authentication = new UsernamePasswordAuthenticationToken(
        decoded.sub(),
        null,
        authorities);
```

#### ✔ Correto — modelo de autoridades (`hasAuthority` / `hasAnyAuthority`)

Com autoridades, use o valor literal, **sem** prefixo, pois `hasAuthority("report:read")` procura exatamente por `report:read`:

```java
List<GrantedAuthority> authorities = decoded.authorities().stream()
        .map(SimpleGrantedAuthority::new)
        .collect(Collectors.toList());

var authentication = new UsernamePasswordAuthenticationToken(
        decoded.sub(),
        null,
        authorities);
```

#### ❌ Incorreto — authority sem prefixo com matcher `hasRole`

```java
// Cria "ADMIN", mas hasRole("ADMIN") procura por "ROLE_ADMIN": nunca casa.
List<GrantedAuthority> authorities = decoded.roles().stream()
        .map(SimpleGrantedAuthority::new)
        .collect(Collectors.toList());
// ...
.requestMatchers("/api/v1/admin/**").hasRole("ADMIN")
```

As seguintes regras devem ser respeitadas:

- As regras devem ser declaradas da mais específica para a mais genérica, sempre encerrando com `anyRequest()`.
- `anyRequest().authenticated()` (ou mais restritivo) deve ser sempre a última regra; nenhum endpoint pode ficar liberado por omissão.
- `permitAll()` deve ser usado apenas em endpoints genuinamente públicos (login, refresh, criação de usuário, documentação, actuator), sempre de forma explícita.
- `authenticated()` exige apenas um usuário autenticado, sem papel específico.
- `hasRole("ADMIN")` / `hasAnyRole("ADMIN", "MANAGER")` exigem um papel. O Spring adiciona o prefixo `ROLE_` automaticamente — declare o papel **sem** o prefixo (`"ADMIN"`, não `"ROLE_ADMIN"`).
- `hasAuthority("task:read")` / `hasAnyAuthority(...)` exigem uma autoridade literal, **sem** prefixo automático. Não misture `hasRole` e `hasAuthority` para expressar o mesmo conceito; escolha um modelo (papéis ou autoridades) e mantenha-o em todo o projeto.
- O método HTTP deve ser restringido quando a autorização depender dele, usando a sobrecarga `requestMatchers(HttpMethod.X, "...")`.
- A autorização por endpoint cobre apenas **categorias** de acesso (público, autenticado, por papel). A verificação de que um recurso pertence ao usuário autenticado é regra de negócio e não pertence aqui — pertence ao caso de uso.

### ✔ Correto

```java
.authorizeHttpRequests(auth -> auth
        // permitAll — público, declarado explicitamente, por método
        .requestMatchers(HttpMethod.POST, "/api/v1/users").permitAll()
        .requestMatchers(HttpMethod.POST, "/api/v1/auth/login").permitAll()
        .requestMatchers(HttpMethod.POST, "/api/v1/auth/refresh").permitAll()
        .requestMatchers("/actuator/**").permitAll()
        .requestMatchers("/docs", "/swagger-ui.html", "/swagger-ui/**", "/v3/api-docs/**").permitAll()
        // denyAll — rota desativada, bloqueada para todos
        .requestMatchers("/api/v1/internal/**").denyAll()
        // hasRole — um único papel (sem o prefixo ROLE_)
        .requestMatchers(HttpMethod.DELETE, "/api/v1/users/**").hasRole("ADMIN")
        // hasAnyRole — qualquer um dos papéis
        .requestMatchers("/api/v1/admin/**").hasAnyRole("ADMIN", "MANAGER")
        // hasAuthority — uma única autoridade literal (sem prefixo)
        .requestMatchers(HttpMethod.GET, "/api/v1/reports/**").hasAuthority("report:read")
        // hasAnyAuthority — qualquer uma das autoridades
        .requestMatchers(HttpMethod.POST, "/api/v1/reports/**").hasAnyAuthority("report:write", "report:admin")
        // authenticated — fecho: todo o restante exige apenas autenticação
        .anyRequest().authenticated())
```

### ❌ Incorreto — `anyRequest()` antes das regras específicas

```java
.authorizeHttpRequests(auth -> auth
        // anyRequest() casa com tudo primeiro: as regras abaixo nunca são avaliadas.
        .anyRequest().authenticated()
        .requestMatchers("/api/v1/admin/**").hasRole("ADMIN"))
```

### ❌ Incorreto — prefixo `ROLE_` manual em `hasRole`

```java
// hasRole já adiciona ROLE_: isto exige a autoridade "ROLE_ROLE_ADMIN" e nunca casa.
.requestMatchers("/api/v1/admin/**").hasRole("ROLE_ADMIN")
```

---

## Autorização por Método

Quando a autorização precisa acompanhar um método específico de serviço ou controller, em vez de uma rota, utiliza-se a segurança por método do Spring (`@PreAuthorize`, `@PostAuthorize`, `@Secured`), habilitada por `@EnableMethodSecurity`.

Este projeto concentra a autorização de acesso no `authorizeHttpRequests`. A segurança por método deve ser adotada apenas quando a regra de acesso **não** puder ser expressa por rota e método HTTP, e mesmo assim nunca deve carregar regra de negócio — para isso existe a verificação no caso de uso.

As seguintes regras devem ser respeitadas:

- `@EnableMethodSecurity` deve ser declarada em uma classe de configuração de segurança, nunca espalhada pela aplicação.
- As anotações (`@PreAuthorize` etc.) expressam apenas categorias de acesso (papel/autoridade), nunca verificação de propriedade de recurso.
- A expressão de autorização deve ser simples e declarativa; lógica condicional sobre o domínio pertence ao caso de uso.
- Prefira a autorização por endpoint no `SecurityConfig`; recorra à segurança por método apenas quando a rota não for suficiente para expressar a regra.

Dentro do `@PreAuthorize`, os matchers são escritos como expressões SpEL — as mesmas condições da autorização por endpoint, agora como texto entre aspas simples. As aspas simples são obrigatórias, pois o argumento externo da anotação já usa aspas duplas.

### ✔ Correto

```java
// hasRole — um único papel (sem o prefixo ROLE_)
@PreAuthorize("hasRole('ADMIN')")
public void deleteAnyUser(String userId) { ... }

// hasAnyRole — qualquer um dos papéis
@PreAuthorize("hasAnyRole('ADMIN', 'MANAGER')")
public void listAllUsers() { ... }

// hasAuthority — uma única autoridade literal (sem prefixo)
@PreAuthorize("hasAuthority('report:read')")
public void readReport(String reportId) { ... }

// hasAnyAuthority — qualquer uma das autoridades
@PreAuthorize("hasAnyAuthority('report:write', 'report:admin')")
public void updateReport(String reportId) { ... }
```

### ❌ Incorreto — aspas duplas na expressão SpEL

```java
// A expressão interna precisa de aspas simples; aspas duplas não compilam.
@PreAuthorize("hasRole("ADMIN")")
public void deleteAnyUser(String userId) { ... }
```

### ❌ Incorreto — regra de negócio dentro da anotação

```java
// Propriedade do recurso é regra de negócio: pertence ao caso de uso, não à anotação.
@PreAuthorize("#task.ownerId == authentication.name")
public void updateTask(TaskEntity task) { ... }
```

---

# Dependências

A implementação de segurança pode depender de:

- Application (para as portas, DTOs e exceções que implementa);
- Spring Security (apenas em `configuration/security/` e `external/security/`);
- Biblioteca de JWT (apenas no `TokenAdapter`);
- Biblioteca de hashing de senha (apenas no `PasswordEncoderAdapter`).

Não é permitido:

- Adicionar dependência de JWT, Spring Security ou de bibliotecas de criptografia nas camadas Domain, Application ou Presentation.
- Acessar propriedades de segurança via `@Value`.
- Expor tipos da biblioteca de JWT (ex.: `Claims`) para fora da camada External.
- Fazer com que qualquer regra de negócio dependa de detalhes de segurança que não sejam as portas.

---

# Resumo das Convenções

Toda implementação de segurança deve respeitar os seguintes princípios:

- A segurança é um detalhe de infraestrutura e o negócio depende apenas de contratos (portas), nunca de JWT ou Spring Security.
- A convenção separa autenticação (quem é o usuário e como o token é emitido/validado) de autorização (o que o usuário autenticado pode acessar).
- O estilo segue o idiomático do Java: `record` para DTOs e propriedades, enums com construtor e getter, `private static final` para constantes e logger, injeção por construtor com campos `final` e tradução de exceção por `try/catch`.

Autenticação:

- `SecurityConfig` e `SecurityConfigProperties` residem em `configuration/security/`; `Token` e `TokenType` residem em `application/port/security/`; `TokenAdapter` reside em `external/security/`.
- `SecurityConfig` usa `@Configuration` + `@EnableWebSecurity`, sessão `STATELESS`, endpoints públicos explícitos e `anyRequest().authenticated()` como fecho.
- `SecurityConfigProperties` é um `record` com `@ConfigurationProperties(prefix = "security")`, sufixo `ConfigProperties`, subgrupo `jwt` como `record` aninhado e valores por variável de ambiente.
- O segredo JWT tem no mínimo 32 caracteres e nunca é versionado com valor real.
- A porta `Token` expressa comportamento de negócio e retorna `DecodeTokenDto` (um `record`), nunca tipos da biblioteca.
- `TokenType` é um enum cujo `value` textual trafega no token, com conversão centralizada (`fromValue`) que lança `InvalidTokenException` para valores desconhecidos.
- `TokenAdapter` implementa `Token` como `@Component`, recebe `SecurityConfigProperties` por construtor, deriva a chave uma única vez no construtor e traduz toda exceção da biblioteca para `InvalidTokenException` via `try/catch`.
- `JwtAuthenticationFilter` estende `OncePerRequestFilter`, depende apenas da porta `Token`, popula o `SecurityContextHolder` quando o token é válido e, quando inválido, segue a cadeia sem bloquear, delegando o `401` ao `authenticationEntryPoint`.
- `InvalidTokenException` estende `RuntimeException`, carrega mensagem fixa e é a única exceção de token conhecida pelo negócio.
- Nenhum dado sensível (token, segredo, senha) deve ser registrado em log.

Autorização:

- A autorização de endpoints é declarada em `authorizeHttpRequests`, da regra mais específica para a mais genérica, encerrando com `anyRequest()` como fecho deny-by-default; `permitAll()` é sempre explícito e `hasRole` é declarado sem o prefixo `ROLE_`.
- A segurança por método (`@PreAuthorize` + `@EnableMethodSecurity`) é adotada apenas quando a rota não expressa a regra, e nunca carrega regra de negócio (propriedade de recurso pertence ao caso de uso).
