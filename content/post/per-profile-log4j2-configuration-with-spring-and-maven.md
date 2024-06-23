+++
title = "Per-profile Log4j2 configuration with Spring Boot and Maven"

date = "2020-10-08"
tags = [
    "maven",
    "spring",
]
+++


A quick guide to configuring many log4j2 configurations with different profiles (QA, Prod…).

First, you need to add the log4j2 spring starter, and to exclude the logback starter (here the version is inherited from your Spring Boot BOM).

```xml
<dependency>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <!-- Exclude logback (default implementation) -->
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-logging</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

You can put your ```log4j2-local.xml``` and the ```log4j2-prod.xml``` in the resource directory of your choice, in this example it will be ```src/main/resources/log/```.

Then, you need to define your profile in your ```pom.xml``` with a property which can be used later as a placeholder:
```xml
<profiles>
    <profile>
        <id>local</id>
        <properties>
            <log4j2.config>log/log4j2-local.xml</log4j2.config>
        </properties>
    </profile>
    <profile>
        <id>prod</id>
        <properties>
            <log4j2.config>log/log4j2-prod.xml</log4j2.config>
        </properties>
    </profile>
</profiles>
```

Finally, you have to add your property placeholder which loads the file from the classpath in your Spring root configuration. It will be replaced at each build with your active profile :)
For exemple in my ```application.yml```:

```yaml
logging:
  config: classpath:${log4j2.config}
```

Note: If you have a specific profile for your unit/integration tests and you want to have a custom log4j2 configuration, the easier way I have found is to put the file in the test resources classpath (```src/test/resources/log/``` for example), thus, it will be taken first by the ```LogManager```.
