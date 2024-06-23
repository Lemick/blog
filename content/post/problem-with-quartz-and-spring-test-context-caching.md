+++
title = "A problem with Quartz and Spring Test Context caching"

date = "2021-06-08"
tags = [
    "quartz",
    "spring-test",
    "spring",
]
+++


When you want to speed up your integration tests, you often want to use ApplicationContext caching. It allows you to reuse your context across tests instead of destroying it at the end of each test, avoiding ~5 seconds of container initialization. You just have to clean your database between each test and Spring will caches all automatically, each Context containerized in its own space.
But when you try to write Quartz tests after this change, some problems appear..

## The Problem with Quartz

Quartz is often used as the default task scheduler in the Spring eco-system, its integration is invisible thanks to the ```spring-boot-starter-quartz```, when configured, the Quartz database calls are delegated to the Spring  ```LocalJobFactory``` with two providers, a "*springNonTxDataSource*" and a "*springTxDataSource*", allowing you to use Quartz in a Spring-managed transaction with @Transactional.

A weakness of Quartz is that it uses a static class when retrieving a connection to the database (```DBConnectionManager```). This brings real problems in the following test case:
* Test 1 creates an ApplicationContext (because no one exists yet) and executes its test, nothing wrong
* Test 2 needs to create another ApplicationContext legitimately (for mocking a web-service call with *@MockBean* for example), when initializing his context, the dependencies are recreated and cannot "leak" from Context 1 to Context 2. The only exception is when the  ```LocalDataSourceJobStore``` is created, a static call is done for providing the correct DataSource to Quartz


```java
DBConnectionManager.getInstance()
    .addConnectionProvider("springTxDataSource." + this.getInstanceName(), new ConnectionProvider() {
        public Connection getConnection() throws SQLException {
            return DataSourceUtils.doGetConnection(LocalDataSourceJobStore.this.dataSource);
        }

        public void shutdown() {
        }

        public void initialize() {
        }
});
```

This call to ```DataSourceUtils.doGetConnection``` is important because it's the class responsible for returning the current JDBC transactional connection when using *@Transactional*.

As you can guess, this static call to the DBConnectionManager overwrites the Datasource created in Context 1.

Now if another test executes with the Context 1 and do something with the Quartz Scheduler in a transaction, the wrong Datasource will be called, resulting in weird troubles, in my case, HikariCP was returning ClosedConnection, and the next lines was throwing the following exception:

```
Caused by: org.quartz.impl.jdbcjobstore.LockException: Failure obtaining db row lock: Connection is closed [See nested exception: java.sql.SQLException: Connection is closed]
```

## My solution

The only way i found to avoid this problem is to register a ```TestExecutionListener``` that implements the  ```beforeTestClass``` method and resets the ```DBConnectionManager``` with the correct providers before each Test.

```java
private void setQuartzConnectionProviders() throws SchedulerException {
    DBConnectionManager.getInstance()
       .addConnectionProvider("springTxDataSource." + scheduler.getSchedulerName(),
            new ConnectionProvider() {
                public Connection getConnection() throws SQLException {
                    return DataSourceUtils.doGetConnection(dataSource);
                }

                public void shutdown() {
                }

                public void initialize() {
                }
            });
    DBConnectionManager.getInstance()
       .addConnectionProvider("springNonTxDataSource." + scheduler.getSchedulerName(),
            new ConnectionProvider() {
                public Connection getConnection() throws SQLException {
                    return dataSource.getConnection();
                }

                public void shutdown() {
                }

                public void initialize() {
                }
            });
}
```


If you are using Spring Test context caching, your test execution must be managed by a single process. So this static call before each test is acceptable (based on the fact that the quartz DBConnectionManager will not be redesigned).
