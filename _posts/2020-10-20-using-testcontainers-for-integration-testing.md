---
layout: post
title:  Using Testcontainers for Integration Testing
description: As a follow on from my last post about running sql server in docker, I thought i’d write about something...
date:   2020-10-20 15:01:35 +0300
image:  '/images/posts/2020-10-20-using-testcontainers-for-integration-testing/testing.png'
tags:   [docker, testing]
---

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">You don&#39;t. <br><br>You test them using integration tests against the actual DB using Docker or <a href="https://twitter.com/testcontainers?ref_src=twsrc%5Etfw">@testcontainers</a></p>&mdash; Vlad Mihalcea (@vlad_mihalcea) <a href="https://twitter.com/vlad_mihalcea/status/1196026201669881857?ref_src=twsrc%5Etfw">November 17, 2019</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

## 👋 Introduction
As a follow on from my last post about running sql server in docker, I thought i’d write about something that I have just introduced for the first time into one of my projects – in the hope that some of you feel as passionate as I do about testing your production code (and not just the equivalent in [H2](https://www.h2database.com/html/main.html)-specific syntax 😱). I hope that through this post I can show you how easy it is to include this dependency into your project, and be on your way to having your integration tests running against your production database!

You can use any resource in your tests that has a docker image (E.g. databases, web browsers, message queues, etc) but in this post I will be outlining how to include testcontainers’ own [MS SQL Server Module](https://www.testcontainers.org/modules/databases/mssqlserver/) alongside [Spock](https://spockframework.org/) to write your integration tests.

All code in this post is available open source under the MIT licence on my [GitHub](https://github.com/MTJB/blog-code-examples), so please feel free to check it out use it as you wish for your own projects.

## 🧐 Requirements
In order to use testcontainers, you will of course need to have Docker [installed and running](https://docs.docker.com/desktop/). You will also need to add the the testcontainer depencies for MSSQLServer, for example (in Gradle);

```java
testCompile 'org.testcontainers:testcontainers:1.14.3'
testCompile 'org.testcontainers:mssqlserver:1.14.3'
```

Since MSSQL Server is a licensed product – you will also need to accept the license or the container will fail to start – to do this simply place a file named [container-license-acceptance.txt](https://github.com/MTJB/blog-code-examples/blob/main/src/test/resources/container-license-acceptance.txt) into test resources folder.

## ✍️ Writing your first test
Now that you have the testcontainer dependency on your classpath, its time to actually use it!

#### Define an Initializer
An Initializer – simply put – is what tells the ApplicationContext where your db is – by nature the Docker container gets a randomly allocated port so this cannot be statically set (or rather, should _not_ be to avoid port collisions on startup).

```java
import org.springframework.context.ApplicationContextInitializer
import org.springframework.context.ConfigurableApplicationContext
import org.springframework.context.annotation.Profile
import org.springframework.core.env.ConfigurableEnvironment
import org.springframework.core.env.PropertiesPropertySource
import org.springframework.test.context.ContextConfiguration
import org.testcontainers.containers.MSSQLServerContainer
import spock.lang.Shared
import spock.lang.Specification
 
import java.sql.Connection
import java.sql.DriverManager
import java.sql.PreparedStatement
 
@Profile("mssql-test")
@ContextConfiguration(initializers = Initializer.class)
class SqlSpec extends Specification {
 
    @Shared
    static MSSQLServerContainer mssqlServerContainer
 
    static class Initializer implements ApplicationContextInitializer<ConfigurableApplicationContext> {
        @Override
        void initialize(ConfigurableApplicationContext configurableApplicationContext) {
 
            mssqlServerContainer = new MSSQLServerContainer()
            mssqlServerContainer.start()
 
            ConfigurableEnvironment environment = configurableApplicationContext.getEnvironment()
 
            String db = environment.getProperty("test.db.name")
            String connectionString = "${mssqlServerContainer.getJdbcUrl()};integratedSecurity=false;username=${mssqlServerContainer.getUsername()};password=${mssqlServerContainer.getPassword()}"
            String jdbcUrl = connectionString + ";databaseName=${db}"
 
            Connection connection = DriverManager.getConnection(connectionString)
            PreparedStatement ps = connection.prepareStatement("IF(db_id(N'${db}') IS NULL) CREATE DATABASE ${db}")
            ps.execute()
 
            Properties props = new Properties()
            props.put("spring.datasource.driver-class-name", mssqlServerContainer.getDriverClassName())
            props.put("spring.datasource.url", jdbcUrl)
 
            environment.getPropertySources().addFirst(new PropertiesPropertySource("myTestDBProps", props))
            configurableApplicationContext.setEnvironment(environment);
        }
    }
}
```

Above you can see how I initialise my ApplicationContext, first I manually start the MSSQL container, I then grab the JDBC url in order for me to set the `spring.datasource.url` – and that’s basically it! I have also added a few extras, like reading the database name from a properties file and creating the database with a PreparedStatement, neither of these are required – and in fact you may want to consider disabling `spring.jpa.hibernate.ddl-auto` entirely as [it is not advised to use in production](https://stackoverflow.com/questions/221379/hibernate-hbm2ddl-auto-update-in-production/221422#221422) – instead opting for some other schema management solution, for example [Flyway](https://vladmihalcea.com/flyway-database-schema-migrations/).

#### Use the test container in a test class
Now that you have defined an `Initializer`, you can use class inheritance for test execution.

```java
@SpringBootTest
@ActiveProfiles("mssql-test")
class SqlserverTests extends SqlSpec {
    // Test functions
}
```

#### Running your tests
With test containers set up for the test suite, running tests is easy – it just works! As you can see in the log output below, test containers takes it from here – It pulls the MSSQL Server image (if it does not exist), starts it, and then starts running your tests.

```java
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v2.3.4.RELEASE)
 
2020-10-20 21:04:33.768  INFO 14778 --- [           main] o.t.d.DockerClientProviderStrategy       : Loaded org.testcontainers.dockerclient.UnixSocketClientProviderStrategy from ~/.testcontainers.properties, will try it first
2020-10-20 21:04:34.339  INFO 14778 --- [           main] o.t.d.UnixSocketClientProviderStrategy   : Accessing docker with local Unix socket
2020-10-20 21:04:34.339  INFO 14778 --- [           main] o.t.d.DockerClientProviderStrategy       : Found Docker environment with local Unix socket (unix:///var/run/docker.sock)
2020-10-20 21:04:34.439  INFO 14778 --- [           main] org.testcontainers.DockerClientFactory   : Docker host IP address is localhost
2020-10-20 21:04:34.470  INFO 14778 --- [           main] org.testcontainers.DockerClientFactory   : Connected to docker: 
  Server Version: 19.03.12
  API Version: 1.40
  Operating System: Docker Desktop
  Total Memory: 1990 MB
2020-10-20 21:04:34.577  INFO 14778 --- [           main] o.t.utility.RegistryAuthLocator          : Credential helper/store (docker-credential-desktop) does not have credentials for index.docker.io
2020-10-20 21:04:34.988  INFO 14778 --- [           main] org.testcontainers.DockerClientFactory   : Ryuk started - will monitor and terminate Testcontainers containers on JVM exit
2020-10-20 21:04:34.988  INFO 14778 --- [           main] org.testcontainers.DockerClientFactory   : Checking the system...
2020-10-20 21:04:34.988  INFO 14778 --- [           main] org.testcontainers.DockerClientFactory   : ✔︎ Docker server version should be at least 1.6.0
2020-10-20 21:04:35.073  INFO 14778 --- [           main] org.testcontainers.DockerClientFactory   : ✔︎ Docker environment should have more than 2GB free disk space
2020-10-20 21:04:35.090  INFO 14778 --- [           main] ?.microsoft.com/mssql/server:2017-CU12]  : Creating container for image: mcr.microsoft.com/mssql/server:2017-CU12
2020-10-20 21:04:35.129  INFO 14778 --- [           main] o.t.utility.RegistryAuthLocator          : Credential helper/store (docker-credential-desktop) does not have credentials for mcr.microsoft.com
2020-10-20 21:04:35.181  INFO 14778 --- [           main] ?.microsoft.com/mssql/server:2017-CU12]  : Starting container with ID: f5b16e1334fbc830620e946267359e522e998238466f279dbf57b33068da7a1a
2020-10-20 21:04:35.428  INFO 14778 --- [           main] ?.microsoft.com/mssql/server:2017-CU12]  : Container mcr.microsoft.com/mssql/server:2017-CU12 is starting: f5b16e1334fbc830620e946267359e522e998238466f279dbf57b33068da7a1a
2020-10-20 21:04:35.438  INFO 14778 --- [           main] ?.microsoft.com/mssql/server:2017-CU12]  : Waiting for database connection to become available at jdbc:sqlserver://localhost:32769 using query 'SELECT 1'
2020-10-20 21:04:35.566  WARN 14778 --- [           main] c.m.s.j.internals.SQLServerConnection    : ConnectionID:1 ClientConnectionId: eb4c676c-2bb2-4c47-8df0-ea0110c3fd0e Prelogin error: host localhost port 32769 Unexpected end of prelogin response after 0 bytes read
2020-10-20 21:04:35.674  WARN 14778 --- [           main] c.m.s.j.internals.SQLServerConnection    : ConnectionID:1 ClientConnectionId: c0fb5082-3b6b-4f27-82fa-d753e7fe0a38 Prelogin error: host localhost port 32769 Unexpected end of prelogin response after 0 bytes read
2020-10-20 21:04:35.879  WARN 14778 --- [           main] c.m.s.j.internals.SQLServerConnection    : ConnectionID:1 ClientConnectionId: b70dfe25-bd4a-4c5a-9975-5decf0969878 Prelogin error: host localhost port 32769 Unexpected end of prelogin response after 0 bytes read
2020-10-20 21:04:36.283  WARN 14778 --- [           main] c.m.s.j.internals.SQLServerConnection    : ConnectionID:1 ClientConnectionId: 049fe67a-0566-490b-88cf-db04e5f67be1 Prelogin error: host localhost port 32769 Unexpected end of prelogin response after 0 bytes read
2020-10-20 21:04:37.090  WARN 14778 --- [           main] c.m.s.j.internals.SQLServerConnection    : ConnectionID:1 ClientConnectionId: 2a6e9f21-ce7d-4552-83e3-bf10d121d4fb Prelogin error: host localhost port 32769 Unexpected end of prelogin response after 0 bytes read
2020-10-20 21:04:38.095  WARN 14778 --- [           main] c.m.s.j.internals.SQLServerConnection    : ConnectionID:1 ClientConnectionId: 68422ef9-cad0-4e55-8908-7ebef0a28e4c Prelogin error: host localhost port 32769 Unexpected end of prelogin response after 0 bytes read
2020-10-20 21:04:39.104  WARN 14778 --- [           main] c.m.s.j.internals.SQLServerConnection    : ConnectionID:1 ClientConnectionId: eb1cbb36-f853-4da8-9f39-e079b2eb908f Prelogin error: host localhost port 32769 Unexpected end of prelogin response after 0 bytes read
2020-10-20 21:04:40.107  WARN 14778 --- [           main] c.m.s.j.internals.SQLServerConnection    : ConnectionID:1 ClientConnectionId: f7cde6b9-1ed2-4b1c-8bc2-cc7969f9907d Prelogin error: host localhost port 32769 Unexpected end of prelogin response after 0 bytes read
2020-10-20 21:04:41.305  INFO 14778 --- [           main] ?.microsoft.com/mssql/server:2017-CU12]  : Container is started (JDBC URL: jdbc:sqlserver://localhost:32769)
2020-10-20 21:04:41.306  INFO 14778 --- [           main] ?.microsoft.com/mssql/server:2017-CU12]  : Container mcr.microsoft.com/mssql/server:2017-CU12 started in PT6.215744S
2020-10-20 21:04:42.181  INFO 14778 --- [           main] c.m.e.TestContainers.SqlserverTests      : Starting SqlserverTests on 127.0.0.1 with PID 14778 (started by mtjb in /blog-code-examples)
```

## 🧪 Conclusion
As you can see – if just a few short steps you can add the testcontainers dependency to your project, and an Initializer, and be on your way to writing your first test against MSSQL Server. At the time of writing this I have only been using testcontainers for a few weeks, but it is already finding more and more uses every day when it comes to automating some of our features – no longer do we have the excuse; “If we do it that way I can’t test that in H2”!