# Convenções de Cache

O **cache** é um detalhe de infraestrutura utilizado para reduzir o acesso a recursos custosos, como o banco de dados.

Nesta arquitetura, toda implementação de cache pertence exclusivamente à camada **External**, por representar um detalhe de infraestrutura.

O cache nunca deve ser conhecido pelas camadas Domain, Application ou Presentation. Nenhuma regra de negócio deve depender da existência, ausência ou comportamento do cache.

Toda implementação utiliza a abstração de cache do Spring (`@EnableCaching`, `@Cacheable`, `@CacheEvict`), o que mantém o código de negócio independente do provedor escolhido. Dois provedores são suportados:

- **Caffeine** — cache local, em memória, na própria JVM. É o padrão para aplicações de instância única.
- **Redis** — cache distribuído, em processo separado. É o padrão para aplicações com múltiplas instâncias, que precisam compartilhar o cache.

Por utilizarem a mesma abstração do Spring, as regras sobre onde aplicar o cache, invalidação, chaves e TTL são idênticas para ambos. As diferenças se restringem à configuração do provedor e à serialização.

---

# Princípios

Toda implementação de cache deve respeitar os seguintes princípios:

- O cache é um detalhe de infraestrutura e reside exclusivamente na camada External.
- O cache é transparente para as demais camadas: a aplicação funciona de forma idêntica com ou sem ele.
- O cache nunca altera o resultado de uma operação, apenas evita o custo de recalculá-la.
- A consistência é prioridade sobre o desempenho: dados desatualizados nunca devem ser servidos após uma escrita.
- O cache é sempre acompanhado de um TTL, que atua como rede de segurança contra falhas de invalidação.
- O TTL é definido por cache, respeitando o ritmo de mudança de cada tipo de dado, com um TTL default para os caches não especificados.
- O cache não deve ser adicionado por precaução. Deve ser introduzido apenas quando houver necessidade comprovada, preferencialmente medida através da observabilidade do projeto.

---

# Escolha do Backend

A escolha entre Caffeine e Redis é uma decisão de infraestrutura e não afeta o código anotado com `@Cacheable`/`@CacheEvict`.

O critério de escolha é o número de instâncias da aplicação:

- **Caffeine** deve ser a escolha padrão para aplicações de instância única. É local à JVM, não exige serialização, não adiciona dependência de infraestrutura externa e possui a menor latência possível.
- **Redis** deve ser utilizado apenas quando a aplicação roda em múltiplas instâncias e o cache precisa ser compartilhado entre elas, ou quando o cache deve sobreviver ao reinício da aplicação.

As seguintes regras devem ser respeitadas:

- A escolha do provedor deve ser justificada pela topologia de implantação, nunca por preferência.
- Não se deve adotar Redis por antecipação. Enquanto a aplicação for de instância única, Caffeine é suficiente.
- Apenas um provedor deve estar ativo por vez. Não se deve configurar Caffeine e Redis simultaneamente como `CacheManager`.
- A troca de provedor não deve exigir alteração nos DAOs, apenas na configuração da camada External.

---

# Troca de Provedor

Como ambos os provedores utilizam a mesma abstração do Spring, a migração entre Caffeine e Redis (em qualquer sentido) é uma alteração restrita à configuração e às dependências. O código de negócio, os DAOs e as anotações de cache permanecem inalterados.

## O que nunca muda na troca

- As anotações `@Cacheable` e `@CacheEvict` nos DAOs.
- Os nomes de cache (`cacheNames`) e as chaves (`key`).
- A estratégia de invalidação.
- A classe de propriedades `CacheConfigProperties` (`defaultTtl` + `Map` de TTLs), pois descreve o comportamento do cache, não o provedor.
- O bloco de propriedades `cache.*` no `application.yaml` (os TTLs por cache).

## O que muda na troca

- A dependência do provedor no `build.gradle.kts`.
- A definição do bean `CacheManager` (ou de suas configurações) na classe `CacheConfig`.
- No caso do Redis: a configuração de conexão (`spring.data.redis.*`) e a definição da serialização JSON, que não existem no Caffeine.

## De Caffeine para Redis

Adotado quando a aplicação passa a rodar em múltiplas instâncias.

1. Substituir a dependência de Caffeine pela de Spring Data Redis no `build.gradle.kts`.
2. Trocar o `CaffeineCacheManager` pela configuração de Redis (`RedisCacheConfiguration` + `RedisCacheManagerBuilderCustomizer`), lendo os mesmos TTLs de `CacheConfigProperties`.
3. Configurar a conexão via `spring.data.redis.*` no `application.yaml`.
4. Configurar a serialização JSON (obrigatória no Redis).
5. Garantir que os Models cacheados sejam serializáveis em JSON (estrutura simples, sem Value Objects).

## De Redis para Caffeine

Adotado quando a aplicação volta a ser de instância única, ou para simplificar a infraestrutura.

1. Substituir a dependência de Spring Data Redis pela de Caffeine no `build.gradle.kts`.
2. Trocar a configuração de Redis pelo `CaffeineCacheManager`, lendo os mesmos TTLs de `CacheConfigProperties`.
3. Remover a configuração de conexão (`spring.data.redis.*`) e a definição de serialização, que deixam de ser necessárias.

---

# Onde Aplicar o Cache

As anotações de cache do Spring (`@Cacheable` e `@CacheEvict`) devem ser aplicadas exclusivamente nos **DAOs** da camada de persistência.

Este é o único ponto autorizado para uso das anotações de cache, pelos seguintes motivos:

- O DAO é uma interface do Spring Data e, portanto, um bean gerenciado, o que permite a interceptação por proxy (AOP) necessária para as anotações funcionarem.
- O DAO opera com **Models** (`@Entity` JPA), que são objetos serializáveis (construtor sem argumentos e campos escalares), evitando os problemas de serialização das entidades de domínio.
- Manter o cache no DAO impede que qualquer detalhe de cache vaze para as camadas Application, Domain ou Presentation.

As seguintes regras devem ser respeitadas:

- As anotações `@Cacheable` e `@CacheEvict` devem ser aplicadas apenas em métodos declarados no DAO.
- O cache nunca deve ser aplicado em casos de uso.
- O cache nunca deve ser aplicado em entidades de domínio.
- O cache nunca deve ser aplicado em controllers.
- O objeto cacheado deve sempre ser um Model da persistência, nunca uma entidade de domínio.

## ✔ Correto

```java
public interface SampleJpaDao extends JpaRepository<SampleJpaModel, UUID> {

    @Cacheable(cacheNames = "samplesById", key = "#id")
    Optional<SampleJpaModel> findById(UUID id);
}
```

## ❌ Incorreto

```java
@Service
public class FindSampleUseCase {

    @Cacheable(cacheNames = "samplesById", key = "#input.id()")
    public SampleOutputDto execute(FindSampleInputDto input) {
        ...
    }
}
```

## ❌ Incorreto

```java
@Entity
@Cacheable
public class SampleJpaModel { }
```

---

# Habilitação do Cache

O cache do Spring deve ser habilitado explicitamente através da anotação `@EnableCaching`.

As seguintes regras devem ser respeitadas:

- `@EnableCaching` deve ser declarada em uma classe de configuração dedicada, localizada em `configuration/cache/`, junto das demais configurações da aplicação.
- Sem `@EnableCaching`, todas as anotações de cache são silenciosamente ignoradas, sem gerar erro.
- A configuração do cache deve permanecer isolada em sua própria classe de configuração, com responsabilidade única.
- A anotação `@EnableCaching` e a estrutura da configuração são idênticas independentemente do provedor. Apenas a definição do `CacheManager` (ou de suas configurações) muda entre Caffeine e Redis.

## ✔ Correto (Caffeine)

Para Caffeine, o TTL por cache é definido através de um `CacheManager` que registra cada cache com sua própria especificação.

```java
@Configuration
@EnableCaching
public class CacheConfig {

    @Bean
    public CacheManager cacheManager(CacheConfigProperties properties) {
        var cacheManager = new CaffeineCacheManager();
        cacheManager.setCacheNames(properties.caches().keySet());

        properties.caches().forEach((name, ttl) ->
                cacheManager.registerCustomCache(name,
                        Caffeine.newBuilder().expireAfterWrite(ttl).build()));

        return cacheManager;
    }
}
```

## ✔ Correto (Redis)

```java
@Configuration
@EnableCaching
public class CacheConfig {

    @Bean
    public RedisCacheConfiguration redisCacheConfiguration(CacheConfigProperties properties) {
        return RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(properties.defaultTtl())
                .disableCachingNullValues()
                .serializeKeysWith(RedisSerializationContext.SerializationPair
                        .fromSerializer(new StringRedisSerializer()));
    }

    @Bean
    public RedisCacheManagerBuilderCustomizer cacheManagerCustomizer(
            CacheConfigProperties properties,
            RedisCacheConfiguration defaultConfig) {

        return builder -> properties.caches().forEach((name, ttl) ->
                builder.withCacheConfiguration(name, defaultConfig.entryTtl(ttl)));
    }
}
```

A configuração por cache deve derivar da configuração default (injetada como `RedisCacheConfiguration`) e apenas sobrescrever o TTL. Como o `RedisCacheConfiguration` é imutável, `entryTtl(ttl)` retorna uma cópia com o novo TTL, preservando o serializer e as demais definições da configuração default.

Nunca parta de `RedisCacheConfiguration.defaultCacheConfig()` ao configurar um cache específico: isso descartaria a serialização JSON e o `disableCachingNullValues` da configuração default, fazendo justamente os caches configurados individualmente caírem na serialização nativa do Java.

## ❌ Incorreto

```java
@Bean
public RedisCacheManagerBuilderCustomizer cacheManagerCustomizer(CacheConfigProperties properties) {
    return builder -> properties.caches().forEach((name, ttl) ->
            builder.withCacheConfiguration(name,
                    RedisCacheConfiguration.defaultCacheConfig().entryTtl(ttl)));
}
```

---

# Serialização

Esta seção se aplica **exclusivamente ao Redis**. O Caffeine armazena a própria referência do objeto em memória, na mesma JVM, e portanto não realiza serialização.

O Redis armazena apenas bytes, portanto todo objeto cacheado precisa ser serializado e desserializado.

As seguintes regras devem ser respeitadas:

- Deve ser utilizada serialização JSON, nunca a serialização nativa do Java.
- A serialização JSON evita a necessidade de implementar `Serializable` e mantém as chaves e valores legíveis no Redis.
- Apenas Models da persistência devem ser serializados, pois possuem estrutura simples (construtor sem argumentos e campos escalares).
- Entidades de domínio nunca devem ser serializadas, pois possuem Value Objects, campos imutáveis e validação no construtor, o que torna a reconstrução frágil e acopla o domínio à infraestrutura.

## ✔ Correto

```java
.serializeValuesWith(RedisSerializationContext.SerializationPair
        .fromSerializer(new GenericJackson2JsonRedisSerializer()));
```

## ❌ Incorreto

```java
public class SampleEntity extends Entity implements Serializable {
    ...
}
```

---

# Invalidação

Toda operação de escrita deve invalidar as entradas de cache afetadas para garantir a consistência dos dados.

A invalidação deve ser **cirúrgica**, removendo apenas as entradas do recurso efetivamente alterado. A remoção de todas as entradas (`allEntries = true`) deve ser evitada, pois desperdiça o cache dos demais recursos.

As seguintes regras devem ser respeitadas:

- Toda operação de escrita (criação, atualização e remoção) deve invalidar o cache correspondente.
- A invalidação deve utilizar a mesma chave utilizada na leitura, garantindo o isolamento por recurso.
- Os nomes de cache (`cacheNames`) do `@Cacheable` e do `@CacheEvict` devem ser idênticos.
- O uso de `allEntries = true` só é permitido quando não houver forma de identificar a entrada específica, e deve ser documentado.
- Quando um método de escrita não possuir naturalmente a chave de invalidação, a assinatura da porta deve ser ajustada para receber o objeto de domínio completo, nunca identificadores soltos e desconexos.

## ✔ Correto

```java
public interface SampleJpaDao extends JpaRepository<SampleJpaModel, UUID> {

    @Cacheable(cacheNames = "samplesById", key = "#id")
    Optional<SampleJpaModel> findById(UUID id);

    @CacheEvict(cacheNames = "samplesById", key = "#model.id")
    SampleJpaModel save(SampleJpaModel model);

    @CacheEvict(cacheNames = "samplesById", key = "#id")
    void deleteById(UUID id);
}
```

## ❌ Incorreto

```java
public interface SampleJpaDao extends JpaRepository<SampleJpaModel, UUID> {

    @CacheEvict(cacheNames = "samplesById", allEntries = true)
    void deleteById(UUID id);
}
```

---

# Chaves de Cache

As chaves devem identificar de forma única e estável o recurso cacheado.

As seguintes regras devem ser respeitadas:

- A chave deve ser derivada de um valor estável e textual (por exemplo, o identificador do recurso).
- A chave deve refletir o critério de identificação do recurso. Um recurso recuperado pelo seu identificador deve utilizar esse identificador como chave.
- Não devem ser utilizados objetos complexos como chave, pois dependem de `hashCode`/`toString` frágeis.
- O nome do cache (`cacheNames`) deve descrever o conteúdo armazenado, nunca o nome do método.

## ✔ Correto

```java
@Cacheable(cacheNames = "samplesById", key = "#id")
Optional<SampleJpaModel> findById(UUID id);
```

## ❌ Incorreto

```java
@Cacheable(cacheNames = "findSample", key = "#id.toString()")
Optional<SampleJpaModel> findById(IdValueObject id);
```

---

# Configurações

Toda configuração de cache deve ser externalizada e tipada.

O TTL deve ser definido **por cache**, nunca através de um único valor global aplicado indistintamente a todos os caches.

Dados diferentes envelhecem em ritmos diferentes: recursos voláteis exigem TTL curto, enquanto recursos estáveis podem utilizar TTL longo. Um TTL único obrigaria a escolher entre desperdiçar o cache dos dados estáveis ou arriscar servir dados voláteis desatualizados.

As seguintes regras devem ser respeitadas:

- Valores como TTL, host e porta não devem ser codificados diretamente na aplicação.
- As configurações específicas do cache devem utilizar `@ConfigurationProperties`.
- A classe de propriedades deve possuir o sufixo `ConfigProperties` e residir em `configuration/cache/`.
- O TTL deve ser configurado por cache, através de um `Map` que associa cada nome de cache (`cacheNames`) ao seu respectivo TTL.
- Deve existir um TTL default, aplicado aos caches não especificados no `Map`.
- As chaves do `Map` de TTLs devem corresponder exatamente aos `cacheNames` utilizados nas anotações.
- A classe de propriedades (`defaultTtl` + `Map` de TTLs) é idêntica para Caffeine e Redis, pois descreve o comportamento do cache, não o provedor.
- A conexão com o Redis deve ser configurada através das propriedades `spring.data.redis.*`. Essa configuração se aplica apenas quando o provedor for Redis; o Caffeine não exige configuração de conexão.
- Todos os valores devem ser controlados por variáveis de ambiente, com valores padrão adequados ao ambiente local.

## ✔ Correto (propriedades, comum aos dois provedores)

```java
@ConfigurationProperties(prefix = "cache")
public record CacheConfigProperties(
        Duration defaultTtl,
        Map<String, Duration> caches
) { }
```

```yaml
cache:
  default-ttl: ${CACHE_DEFAULT_TTL:60s}
  caches:
    samplesById: ${SAMPLES_CACHE_TTL:30s}
    usersByUsername: ${USERS_CACHE_TTL:5m}
```

## ✔ Correto (conexão adicional, apenas para Redis)

```yaml
spring:
  data:
    redis:
      host: ${REDIS_HOST:localhost}
      port: ${REDIS_PORT:6379}
```

## ❌ Incorreto

```java
@ConfigurationProperties(prefix = "cache")
public record CacheConfigProperties(
        Duration ttl
) { }
```

## ❌ Incorreto

```java
@Configuration
public class CacheConfig {

    private static final long TTL = 60000L;
    private static final String HOST = "localhost";
}
```

---

# Dependências

A implementação de cache pode depender de:

- Application (para os Models e o fluxo de persistência);
- Spring Cache;
- Caffeine (quando o provedor for Caffeine);
- Spring Data Redis e Redis (quando o provedor for Redis).

Deve ser adicionada apenas a dependência do provedor efetivamente utilizado. Não se deve manter Caffeine e Redis no classpath simultaneamente sem necessidade.

Não é permitido:

- Adicionar dependência de cache nas camadas Domain, Application ou Presentation.
- Expor Models cacheados para fora da camada External.
- Fazer com que qualquer regra de negócio dependa do cache.

---

# Resumo das Convenções

Toda implementação de cache deve respeitar os seguintes princípios:

- O cache pertence exclusivamente à camada External.
- O cache é transparente e a aplicação deve funcionar de forma idêntica sem ele.
- O provedor é escolhido pela topologia de implantação: Caffeine para instância única, Redis para múltiplas instâncias.
- A escolha do provedor não deve exigir alteração nos DAOs, apenas na configuração.
- As anotações `@Cacheable` e `@CacheEvict` devem ser aplicadas exclusivamente nos DAOs.
- O cache deve sempre armazenar Models da persistência, nunca entidades de domínio.
- `@EnableCaching` deve ser declarada em uma classe de configuração dedicada em `configuration/cache/`.
- A serialização em JSON aplica-se apenas ao Redis; o Caffeine armazena o objeto em memória, sem serialização.
- Toda operação de escrita deve invalidar o cache de forma cirúrgica, evitando `allEntries = true`.
- Os nomes de cache do `@Cacheable` e do `@CacheEvict` devem ser idênticos.
- Quando faltar a chave de invalidação, a porta deve receber o objeto de domínio completo, nunca identificadores soltos.
- Toda configuração deve ser externalizada e tipada via `@ConfigurationProperties`, com sufixo `ConfigProperties`.
- O cache sempre deve possuir um TTL como rede de segurança, definido por cache através de um `Map`, com um TTL default para os caches não especificados.
- O cache não deve ser adicionado por precaução, apenas mediante necessidade comprovada.
