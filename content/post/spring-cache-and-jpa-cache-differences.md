+++
title = "Spring Cache and JPA Cache differences"
description = "Spring Cache and JPA Cache differences"

date = "2022-03-17"
tags = [
    "spring",
    "jpa",
]
+++

A cache is a good way of optimizing slow calls. The two common caches used in most Spring applications are Spring Cache and the JPA Cache. There are already a lot of very good articles on how to configure each one, but I want to emphasize on their differences.

These caches are independent and configured differently, but they are both compatible with [JCache (JSR-107)](https://www.jcp.org/en/jsr/detail?id=107) specification and can use the same cache implementation as EhCache.

>The JCACHE specification standardizes in process caching of Java objects in a way that allows an efficient implementation, and removes from the programmer the burden of implementing cache expiration, mutual exclusion, spooling,  and cache consistency.

### Spring Cache

Spring Cache is an application-level cache, like many other Spring features, it works by intercepting the call to a bean method with an AOP proxy and handles the need to call the real method (ex: calling an external service with HTTP, reading a file from an SFTP server), and to return the object from the cache instead. 

![Spring cache](/spring-cache.png)

This interceptor is automatically registered when you add the Spring ```@Cacheable``` or the JCache ```@CacheResult``` annotation to a bean method:

```java
@Cacheable(value = "blogPostCache")
public BlogPost fetchBlogPost(Long id) {
        return httpService.fetchFromServer(id);
    }
}
```

Here, the first call to this method will populate the cache with the "id" parameter as the cache entry key, and the next calls will hit the cache until this entry is evicted.

## How to configure Spring Cache

Here is the minimal configuration for a working cache with an externalized configuration:

*src/main/resources/application.yml:*
```yml
spring:
  cache:
    jcache:
      config: classpath:spring-cache.xml
```
We specify here only the EhCache configuration file path, the property ```spring.cache.jcache.provider``` could be specified if you have many ```javax.cache.spi.CachingProvider``` implementations on your classpath (Spring  will warn you in that case), in our case, adding EhCache 3 on our classpath also bring the CachingProvider that will be discovered.


*src/main/resources/spring-cache.xml:*

```xml
<?xml version="1.0" encoding="UTF-8"?>
<config xmlns:xsi='http://www.w3.org/2001/XMLSchema-instance'
        xmlns='http://www.ehcache.org/v3'
        xsi:schemaLocation="http://www.ehcache.org/v3 http://www.ehcache.org/schema/ehcache-core-3.0.xsd">

    <cache alias="blogPostCache">
        <key-type>java.lang.Long</key-type>
        <value-type>com.lemick.app.model.entity.BlogPost</value-type>
        <expiry>
            <tti unit="hours">1</tti>
        </expiry>
    </cache>

</config>

```

We configured the cache "blogPostCache" with the value type it holds, the key type, and an expiration policy of 1 hour.

### JPA Cache

JPA L2C Cache (second-level-cache) is a data access layer cache, its goal is to *avoid fetching an identified entity from the database*, but it can also fetch its associations. It is managed through EntityManagerFactory, so you only have one in a typical application (divided into many cache regions), and it is accessible by any EntityManager instance since it is thread-safe.
The eviction is more complex than a traditional cache because in certain cases it must be synchronized, for example, if you update an entity, you want it to be removed from your cache, and it must be consistent across all your transactions. Entities that are *often read and rarely updated* are good candidates to be cached.

![JPA cache](/jpa-cache.png)

Only identified entity fetched by the EntityManager can be cached directly in L2C. JPA lets you also define a query cache that can help you cache JPQL queries. It depends on L2C, but [it's trickier to use](https://vladmihalcea.com/hibernate-query-cache-n-plus-1-issue/). 

*You never want to store an entity with Spring Cache*. An entity is a mutable object and carrying it across multiple transactions can lead to problems. Moreover, Spring cache knows nothing about all the events applied to your entities, and even JPA does not store your entity in the cache as a POJO but in his disassembled state, which is an Object[] array.

## How to configure JPA Cache

Note: A full-working example with a configured cache can be found [here](https://github.com/Lemick/demo-hibernate-query-utils)

You might already have Hibernate and the EhCache dependency. You'll need to add the ```hibernate-jcache``` dependency since we're using JCache in this example:

```xml
<dependency>
    <groupId>org.hibernate</groupId>
    <artifactId>hibernate-jcache</artifactId>
</dependency>
```

A minimal configuration file can look like this:
*src/resources/application.yml:*


```yaml
spring:
  jpa:
    properties:
      hibernate:
        cache:
          region.factory_class: org.hibernate.cache.jcache.JCacheRegionFactory
          use_second_level_cache: true
          use_query_cache: true
        javax.cache:
          uri: classpath:hibernate-cache.xml
          provider: org.ehcache.jsr107.EhcacheCachingProvider
```
Implicitly Hibernate will bootstrap automatically a ```JCacheRegionFactory``` that will use ```EhcacheCachingProvider``` if this is your only ```CachingProvider``` JCache implementation on the classpath, hence ```cache.region.factory_class``` and ```javax.cache.provider``` could be omitted in this case.
Hibernate disables second-level-cache by default, this is not the case for every JPA provider.

Finally, your configuration file can look like this:
*src/resources/hibernate-cache.xml:*
```xml
<?xml version="1.0" encoding="UTF-8"?>
<config xmlns:xsi='http://www.w3.org/2001/XMLSchema-instance'
        xmlns='http://www.ehcache.org/v3'
        xsi:schemaLocation="http://www.ehcache.org/v3 http://www.ehcache.org/schema/ehcache-core-3.0.xsd">

    <cache alias="com.lemick.app.model.entity.CommentEntity">
        <expiry>
            <ttl unit="minutes">30</ttl>
        </expiry>
    </cache>
</config>
```

We declare our ```CommentEntity``` is ```javax.persistence.Cacheable```, with Hibernate you must also supply a CacheStrategy with the ```Cache``` annotation. You need to set it accordingly to your consistency needs. Here the ```READ_WRITE``` option will put a soft lock on the cache entry if an entity update is detected, avoiding another client to read a stale entity. But there is also a ```READ``` option for immutable entities, a ```NONSTRICT_READ_WRITE``` option when you can assume an eventual consistency, and  a ```TRANSACTIONAL``` option when you need high consistency.

```java
@Entity
@Cacheable
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class CommentEntity {
    private String content;
    
    ...
}
```

When retrieving an entity, the second call will hit directly the cache:
```java
@Transactional
public CommentEntity loadFirstComment() {
    return entityManager.find(CommentEntity.class, 1L);
}

public loadCommentEntity() {
    CommentEntity commentEntity = loadFirstComment(); // Will load from Datasource and populate L2C
    CommentEntity commentEntity2 = loadFirstComment(); // Will load from L2C without hitting the Datasource
}
```

### Conclusion
These two caches use the same specification and can use the same implementation. But their goals are different as they operate on different application layers. Once you know them, you will understand which cache corresponds to each use case.