---
layout: post
title:  "Customize Hikari connection pooling for SpringBoot application"
date:   2021-06-28 18:41:33 +0530
categories: programming
published: true
---

* TOC
{:toc}
# The problem statement

When we try to connect to a database from our Spring boot application, we use connection pooling for creating and maintaining connections. SpringBoot provides native support for [Hikari connection pooler](https://github.com/brettwooldridge/HikariCP) today.

Before SpringBoot 2.x SpringBoot didn't use to support Hikari out of the box. Because of this we needed to add Hikari as an dependency and add proper config values in `application.properties` file. 

After SpringBoot 2.x being introduced and it providing out of the box support for Hikari, we no longer need to special applications properties. We will be able to use the ones [provided by Spring](https://docs.spring.io/spring-boot/docs/current/reference/html/application-properties.html#application-properties.data.spring.datasource.hikari) by default. 

One of the problems I have faced with this approach is that sometimes SpringBoot does not pick up the properties even when the properties are correctly defined. This maybe due to multiple factors like Hikari version that is imported or simply SpringBoot skipping the properties. When the connection pool misbehaves and does not take up the values proved in the application configuration, it creates a lot of frustration. 

![](/eleanor/assets/images/GitHub-Mark-32px.png) **Code Example**

This article is accompanied by a working code example [on GitHub](https://github.com/Gouravmoy/hikariConfig).

# Defining Custom Data source Bean

Because of this, it makes sense to have a better degree of control over the Hikari configuration values ourselves than leaving it up completely for Spring to manage. 

> The below setup is tested with SpringBoot version 2.1.5 onward.

## Adding Common pooling

Add the below dependency in your application

```xml
<dependency>
	<groupId>org.apache.commons</groupId>
	<artifactId>commons-pool2</artifactId>
</dependency>
```

## Application configuration properties

We are going to define some custom application properties so as to not overlap with Spring's default properties

```properties
###########################################################################
# Data Source Configuration
###########################################################################

app.datasource.cp.url=${datasource_url}
app.datasource.cp.username=${datasource_username}
app.datasource.cp.password=${datasource_password}
app.datasource.cp.driverClassName=${datasource_driver_class_name}

###########################################################################
# HikariCP Configuration
###########################################################################

app.datasource.cp.maxConTime=30d
app.datasource.cp.maximumPoolSize=10
app.datasource.cp.poolName=customConnectionPoolName
app.datasource.cp.minPoolSize=2
app.datasource.cp.idleConTimeout=45m

############################################################################
# Application Logging setting
############################################################################
logging.level.com.zaxxer.hikari.HikariConfig=DEBUG 
logging.level.com.zaxxer.hikari=TRACE
```

The **Data Source Configuration** section of the properties defines the basic database connection parameters like the URL, authentication details like user name and password and the database's driver name

## Adding Custom Data Source

So as to have control over the way Spring creates the data source, we need to define our own Data Source Config. This config will be extending Hikari Config to set the pooling attributes

```java
import java.time.Duration;

import javax.sql.DataSource;

import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;

import lombok.Getter;
import lombok.Setter;

@Configuration
@ConfigurationProperties(prefix = "app.datasource.cp")
@Getter
@Setter
public class DataSourceConfig extends HikariConfig {

	private String url;
	private String username;
	private String password;
	private Duration maxConTime;
	private int maximumPoolSize;
	private String poolName;
	private String driverClassName;
	private int minPoolSize;    
	private Duration idleConTimeout;

	@Bean(name = "dataSource")
	public DataSource dataSource() {
		HikariDataSource hikariDataSource = new HikariDataSource();
		hikariDataSource.setJdbcUrl(url);
		hikariDataSource.setUsername(username);
		hikariDataSource.setPassword(password);
		hikariDataSource.setMaximumPoolSize(maximumPoolSize);
		hikariDataSource.setMaxLifetime(maxConTime.toMillis());
		hikariDataSource.setPoolName(poolName);
		hikariDataSource.setIdleTimeout(idleConTimeout.toMillis());
		hikariDataSource.setMinimumIdle(minPoolSize);
		hikariDataSource.setDriverClassName(driverClassName);
		hikariDataSource.setAutoCommit(true);
		return hikariDataSource;
	}
}
```

> `maxLifetime` takes in input as milliseconds. So convert the duration value to milliseconds

Once these values are set, run the application. In the logs, Hikari will log the connection parameters. Check the values to confirm

# Understanding Hikari Connection pooling Configurations 

## Connection configurations

```properties
###########################################################################
# Data Source Configuration
###########################################################################

app.datasource.cp.url=${datasource_url}
app.datasource.cp.username=${datasource_username}
app.datasource.cp.password=${datasource_password}
app.datasource.cp.driverClassName=${datasource_driver_class_name}
```

- `app.datasource.cp.url`- URL to the database 
- `app.datasource.cp.username`- User name of the database
- `app.datasource.cp.password`- Password for connecting to the database
- `app.datasource.cp.driverClassName`- Driver class name

## Connection pooling configurations

```properties
###########################################################################
# HikariCP Configuration
###########################################################################

app.datasource.cp.maxConTime=30d
app.datasource.cp.maximumPoolSize=10
app.datasource.cp.poolName=customConnectionPoolName
app.datasource.cp.minPoolSize=2
app.datasource.cp.idleConTimeout=45m
```



The **Hikari  pooling configuration** section of the properties contains information regarding the connection pooling

- `app.datasource.cp.maxConTime`-  Global connection timeout. This is the time duration after which all connections will be terminated irrespective of state. This value should be set as high as the database infrastructure allows

- `app.datasource.cp.maximumPoolSize` - Maximum number of connections that the pool can create. The number is reached in case of high load but never exceeds the value. The value can be accurately be determined by determining the number of API calls happening to the application and how many database connection would be required at the same time.

  I have kept this value at 10 connections. It can be changed and increased as per requirement.

- `app.datasource.cp.minPoolSize`- Minimum number of connections that is always maintained by the pool irrespective of state of the connections.

- `app.datasource.cp.idleConTimeout` - The time duration after which an idle connection is terminated. This parameter is helpful after a sudden load surge to the application. During the load surge, new connections are created to accommodate the increased database calls. Once the load goes back to normal, the additional connections go back to being idle and are removed after the `idleConTimeout` duration.

## Logging configurations

```properties
############################################################################
# Application Logging setting
############################################################################
logging.level.com.zaxxer.hikari.HikariConfig=DEBUG 
logging.level.com.zaxxer.hikari=TRACE
```

The **Application Logging Settings** section has properties for debugging the connection pooling. This will help to generate Hikari connection pooling information that Spring picks up while you start the application like below

```
com.zaxxer.hikari.HikariConfig:customConnectionPoolName - configuration:
com.zaxxer.hikari.HikariConfig:allowPoolSuspension.............false
com.zaxxer.hikari.HikariConfig:autoCommit......................true
com.zaxxer.hikari.HikariConfig:catalog.........................none
com.zaxxer.hikari.HikariConfig:connectionInitSql...............none
com.zaxxer.hikari.HikariConfig:connectionTestQuery.............none
com.zaxxer.hikari.HikariConfig:connectionTimeout...............30000
com.zaxxer.hikari.HikariConfig:dataSource......................none
com.zaxxer.hikari.HikariConfig:dataSourceClassName.............none
com.zaxxer.hikari.HikariConfig:dataSourceJNDI..................none
com.zaxxer.hikari.HikariConfig:dataSourceProperties............{password=<masked>}
com.zaxxer.hikari.HikariConfig:driverClassName................."org.postgresql.Driver"
com.zaxxer.hikari.HikariConfig:healthCheckProperties...........{}
com.zaxxer.hikari.HikariConfig:healthCheckRegistry.............none
com.zaxxer.hikari.HikariConfig:idleTimeout.....................600000
com.zaxxer.hikari.HikariConfig:initializationFailTimeout.......1
com.zaxxer.hikari.HikariConfig:isolateInternalQueries..........false
com.zaxxer.hikari.HikariConfig:jdbcUrl.........................jdbc:hostname:5432/dbName?sslmode=require
com.zaxxer.hikari.HikariConfig:leakDetectionThreshold..........0
com.zaxxer.hikari.HikariConfig:maxLifetime.....................2592000000
com.zaxxer.hikari.HikariConfig:maximumPoolSize.................10
com.zaxxer.hikari.HikariConfig:metricRegistry..................none
com.zaxxer.hikari.HikariConfig:metricsTrackerFactory...........none
com.zaxxer.hikari.HikariConfig:minimumIdle.....................10
com.zaxxer.hikari.HikariConfig:password........................<masked>
com.zaxxer.hikari.HikariConfig:poolName........................"customConnectionPoolName"
com.zaxxer.hikari.HikariConfig:readOnly........................false
com.zaxxer.hikari.HikariConfig:registerMbeans..................false
com.zaxxer.hikari.HikariConfig:scheduledExecutor...............none
com.zaxxer.hikari.HikariConfig:schema..........................none
com.zaxxer.hikari.HikariConfig:threadFactory...................internal
com.zaxxer.hikari.HikariConfig:transactionIsolation............default
com.zaxxer.hikari.HikariConfig:username........................"username"
com.zaxxer.hikari.HikariConfig:validationTimeout...............5000
```

# Global Connection timeout flow

Global Connection time out is considered as the amount of time after which all the connections in the pool, irrespective of the connection being active or idle or waiting are terminated

This value is set in the parameter - `maxConTime` .

For example, if the `maxConTime` is set to a value of 2 mins (120000ms), then all the connections irrespective of their state will be disconnected and new connections will be made

**What happens after disconnections?**

After all the connections are disconnected, the Hikari will create new connections to fill the pool. The number of connections created will depend upon the load on the system.

Assuming the load to be less, then Hikari will create the the minimum number of connections required to maintain connectivity. This is determined by the parameter `minPoolSize`. This is the bare minimum number of connections Hikari will maintain with the database irrespective of the load on the system.

Assuming the load to be high, then Hikari will keep creating new connections beyond the minimum value set. It will keep creating the connections until it reaches the maximum number of connections that it can create which is determined by the parameter`maximumPoolSize`. No matter how much the load is, it will not create any more connections than this value. It will reuse these connections as per requirement.

**For Example:**

Lets considering the value set in `minPoolSize` is 2 and `maximumPoolSize` is 10,

If the global connection timeout occurs and all existing connections are terminated, then Hikari will try to create new connections. If the load on the system is not too high, then Hikari will create 2 new connections. If the load is high, then Hikari will keep adding more number of connections until the max number of connections is reached, that is 10.

# Idle Connection Timeout flow

Idle connection timeout is considered the time after which an idle connection will be terminated. A connection is termed idle if the connection is active (connected to the database) but is not being used to fetch information from the database.

This value is set in the parameter - `idleConTimeout`

For example, if the `idleConTimeout` is set for 1 min (60000ms), it means that if a connection is idle for 1min, the connection is terminated.

The value of this parameter can be made as high as `maxConTime` or set to `0` (infinite, does not disconnect).

**What happens when a connection is idle for too long?**

If a connection is idle for time equals to the `idleConTimeout` , then the connection/s start getting terminated. This is done so as to return the un-used connection back to the database infrastructure's pool

The terminations of idle connections happens till the number of connections remaining are equal to `minPoolSize`. Connection termination due to idle connections never happen below the minimum value set in the parameter.

**For Example:**

Lets considering we have at present 10 connections available all of which are idle. The value set in `minPoolSize` is 2 and `idleConTimeout` is set at 1 min

Once the connections are idle for 1 minute, Hikari will start terminating the connections till it reaches the value of 2 which is set as the minimum

> Important: Global connection timeout always takes preference to idle connection timeout. This means that even though the connection may not have reached the idle timeout, if the global timeout is reached, then the connection is terminated and new connections are created.

# Conclusion

With the above changes, we can exert better control over how a SpringBoot application handles connections to the database. 
