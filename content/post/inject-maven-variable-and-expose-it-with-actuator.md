+++
title = "Injecting version number with Maven and expose it with Spring Actuator"

date = "2020-09-26"
tags = [
    "maven",
    "actuator",
]
+++

In my current project, we needed to identify each build with a version number. We decided to inject this number at compile-time, so it would be generated at each Jenkins Build with a unique build-number, allowing us to always be in a "production-ready" state.

### Setting up the POM

You can set a default value in the properties, here I've added metadata information "-dev", that is useful to distinguish different build profiles (local, Jenkins CI, etc…).

And it's done! You can now easily inject your custom app-version with the Maven command line, and your variable is ready to be exposed.

You could even inject a variable from your favorite CI tool into that command:

```xml
<properties>
	<app.version>0.0.1-dev</app.version>
</properties>
```

Next, you need to specify your ```app.version``` property in the ```spring-boot-maven-plugin``` so it will be added to the ```build-info.properties``` during the construction of your jar.

```xml
<build>
  <plugins>
      <plugin>
          <groupId>org.springframework.boot</groupId>
          <artifactId>spring-boot-maven-plugin</artifactId>
          <version>${springboot.version}</version>
          <executions>
              <execution>
                  <id>build-info</id>
                  <goals>
                      <goal>build-info</goal>
                  </goals>
                  <configuration>
                      <additionalProperties>
                          <app.version>${app.version}</app.version>
                      </additionalProperties>
                  </configuration>
              </execution>
          </executions>
      </plugin>
  </plugins>
</build>
```

And it's done! You can now easily inject your custom app-version with the Maven command line, and your variable is ready to be exposed.

```shell
mvn install -Dapp.version=1.0.0-60
```

You could even inject a variable from your favorite CI tool into that command:

```shell
mvn install -Dapp.version=0.0.1-${BUILD_NUMBER}
```

### Exposing it with Actuator

You just have to [add the actuator dependency to your project](https://docs.spring.io/spring-boot/docs/current/reference/html/production-ready-features.html) and ```build-info.properties``` will be automatically exposed via the   ``` /actuator/info``` endpoint:

```json
{
  "build" : {
    "app" : {
      "version" : "0.0.1-362"
    },
    "version" : "0.0.1",
    "artifact" : "myApp",
    "name" : "myApp",
    "time" : "2020-09-26T17:34:58.208Z",
    "group" : "com.mk"
  }
}
```

Note: For the project needs, I've used a different property than the conventional "version" property, but you can use it the same way as the example above.
