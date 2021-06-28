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

We are going to define custom application properties so as to not overlap with Spring's default properties

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

############################################################################
# Application Logging setting
############################################################################
logging.level.com.zaxxer.hikari.HikariConfig=DEBUG 
logging.level.com.zaxxer.hikari=TRACE
```

The **Data Source Configuration** section of the properties defines the basic database connection parameters like the URL, authentication details and the database's driver name

The **HikariCP Configuration** section of the properties contains information regarding the connection pooling

- `app.datasource.cp.maxConTime`- this represents the maximum time a connection can stay alive between the application and the database. It is recommended to keep this value as high as the infrastructure in the database side allows. We have provided a value of `30d` or 30 days

- `app.datasource.cp.maximumPoolSize` - this represents the number of concurrent connections that the application can make to the database. The value can be accurately be determined by determining the number of API calls happening to the application and how many database connection would be required at the same time.

  I have kept this value at 10 connections. It can be changed and increased as per requirement.

- `app.datasource.cp.poolName` - this attribute is used to name the pool. This is helpful debuggin and logging purpose.

  The other parameters like `min idle`, `max_idle`are not mandatorily needed to be set as their default values are good enough for the pooling to work.

The **Application Logging Settings** section has properties for debugging the connection pooling. This will help to generate Hikari connection pooling information that Spring picks up while you start the application like below

```
com.zaxxer.hikari.HikariConfig           : customConnectionPoolName - configuration:
com.zaxxer.hikari.HikariConfig           : allowPoolSuspension.............false
com.zaxxer.hikari.HikariConfig           : autoCommit......................true
com.zaxxer.hikari.HikariConfig           : catalog.........................none
com.zaxxer.hikari.HikariConfig           : connectionInitSql...............none
com.zaxxer.hikari.HikariConfig           : connectionTestQuery.............none
com.zaxxer.hikari.HikariConfig           : connectionTimeout...............30000
com.zaxxer.hikari.HikariConfig           : dataSource......................none
com.zaxxer.hikari.HikariConfig           : dataSourceClassName.............none
com.zaxxer.hikari.HikariConfig           : dataSourceJNDI..................none
com.zaxxer.hikari.HikariConfig           : dataSourceProperties............{password=<masked>}
com.zaxxer.hikari.HikariConfig           : driverClassName................."org.postgresql.Driver"
com.zaxxer.hikari.HikariConfig           : healthCheckProperties...........{}
com.zaxxer.hikari.HikariConfig           : healthCheckRegistry.............none
com.zaxxer.hikari.HikariConfig           : idleTimeout.....................600000
com.zaxxer.hikari.HikariConfig           : initializationFailTimeout.......1
com.zaxxer.hikari.HikariConfig           : isolateInternalQueries..........false
com.zaxxer.hikari.HikariConfig           : jdbcUrl.........................jdbc:hostname:5432/dbName?sslmode=require
com.zaxxer.hikari.HikariConfig           : leakDetectionThreshold..........0
com.zaxxer.hikari.HikariConfig           : maxLifetime.....................2592000000
com.zaxxer.hikari.HikariConfig           : maximumPoolSize.................10
com.zaxxer.hikari.HikariConfig           : metricRegistry..................none
com.zaxxer.hikari.HikariConfig           : metricsTrackerFactory...........none
com.zaxxer.hikari.HikariConfig           : minimumIdle.....................10
com.zaxxer.hikari.HikariConfig           : password........................<masked>
com.zaxxer.hikari.HikariConfig           : poolName........................"customConnectionPoolName"
com.zaxxer.hikari.HikariConfig           : readOnly........................false
com.zaxxer.hikari.HikariConfig           : registerMbeans..................false
com.zaxxer.hikari.HikariConfig           : scheduledExecutor...............none
com.zaxxer.hikari.HikariConfig           : schema..........................none
com.zaxxer.hikari.HikariConfig           : threadFactory...................internal
com.zaxxer.hikari.HikariConfig           : transactionIsolation............default
com.zaxxer.hikari.HikariConfig           : username........................"username"
com.zaxxer.hikari.HikariConfig           : validationTimeout...............5000
com.zaxxer.hikari.HikariDataSource       : customConnectionPoolName - Starting...
com.zaxxer.hikari.pool.HikariPool        : customConnectionPoolName - Added connection org.postgresql.jdbc.PgConnection@47162173
com.zaxxer.hikari.HikariDataSource       : customConnectionPoolName - Start completed.
com.zaxxer.hikari.pool.HikariPool        : customConnectionPoolName - Pool stats (total=1, active=1, idle=0, waiting=0)
com.zaxxer.hikari.pool.HikariPool        : customConnectionPoolName - Added connection org.postgresql.jdbc.PgConnection@7856e1a5

```

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

	@Bean(name = "dataSource")
	public DataSource dataSource() {
		HikariDataSource hikariDataSource = new HikariDataSource();
		hikariDataSource.setJdbcUrl(url);
		hikariDataSource.setUsername(username);
		hikariDataSource.setPassword(password);
		hikariDataSource.setMaximumPoolSize(maximumPoolSize);
		hikariDataSource.setMaxLifetime(maxConTime.toMillis());
		hikariDataSource.setPoolName(poolName);
		hikariDataSource.setDriverClassName(driverClassName);
		hikariDataSource.setAutoCommit(true);
		return hikariDataSource;
	}
}
```

> `MaxLifetime` takes in input as milliseconds. So convert the duration value to milliseconds

Once these values are set, run the application. In the logs, Hikari will log the connection parameters. Check the values to confirm

# Conclusion

With the above changes, we can exert better control over how a SpringBoot application handles connections to the database
