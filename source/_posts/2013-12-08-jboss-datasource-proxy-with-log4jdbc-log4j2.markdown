---
layout: post
title: "JBoss 7 Database Tracing"
date: 2013-12-08 18:19:00 +0100
comments: true
published: true
categories:
- jboss
- database
---
When developing applications using databases it can be interesing to know
what is happening at the JDBC level exactly.
In this post a solution and integration into JBoss AS 7.2 is shown.
<!-- more -->
There are many reasons for that:

 * Wanting to know what exactly your ORM framework of choice is doing
 * Verifying correct transaction management
 * Measuring execution time

To help with that datasource [proxies](http://sourceforge.net/projects/p6spy/) [were](https://code.google.com/p/log4jdbc/) [created](https://code.google.com/p/log4jdbc-remix/) that can log those activities you are interested in.
After all these projects [log4jdbc-log4j2](https://code.google.com/p/log4jdbc-log4j2/) includes all the features the others had and even extends on them.
For the goal of this post the possibility to be configured as a datasource is the most important one. This is needed to integrate with container managed datasources.

The following steps will be necessary to enable JDBC tracing for JBoss AS datssources:

 * Add a JBoss AS module for log4jdbc-log4j2
 * Add a datasource driver to the JBoss configuration
 * Adjust the JDBC URL
 * Configure log4jdbc-log4j2 logging in JBoss AS

log4jdbc-log4j2 provides different artifacts for different JDBC versions. For Java 7, which is a good fit for JBoss AS 7.2, the [newest jdbc4.1 version](https://code.google.com/p/log4jdbc-log4j2/downloads/list?q=jdbc4.1) should be used (1.15 when writing this article).


## Add a JBoss AS module for log4jdbc-log4j2

JBoss AS is built upon [JBoss Modules](https://docs.jboss.org/author/display/MODULES/Introduction) which provides a modular (hierarchical) class loading and execution environment. Each module only sees those modules it has registered dependencies for. Every part of JBoss AS is part of a module, even the deployed applications built a module hierarchy. JBoss AS detects the correct dependencies of these applications to container provided modules and adds them dynamically as described in [Implicit module dependencies for deployments](https://docs.jboss.org/author/display/AS72/Implicit+module+dependencies+for+deployments).

log4jdbc-log4j2 also needs to be packaged as a module to be usable by other JBoss AS modules. First the module structure needs to be created including the jar file downloaded before:

``` bash Create module structure
mkdir -p modules/net/sf/log4jdbc/main
cp ~/Downloads/log4jdbc-log4j2-jdbc4.1-1.15.jar modules/net/sf/log4jdbc/main
```

Second a module descriptor is needed at `modules/net/sf/log4jdbc/main/module.xml`:
``` xml module.xml
<module xmlns="urn:jboss:module:1.1" name="net.sf.log4jdbc">
    <resources>
        <resource-root path="log4jdbc-log4j2-jdbc4.1-1.15.jar"/>
    </resources>
    <dependencies>
        <module name="javax.api"/>
        <module name="javax.transaction.api"/>
        <module name="org.slf4j"/>
        <module name="com.h2database.h2" optional="true"/>
        <!-- TODO: add other database modules as other optional dependencies like h2 -->
    </dependencies>
</module>
```

The lines of this [module descriptor](https://docs.jboss.org/author/display/MODULES/Module+descriptors) need explanation. In the first line the `name` of the module is specified so other modules will be able to depend on it. The `resources` section lists the java libraries and classes of the module, path is relative to the module.xml. The `depdendencies` section lists the modules this module depends on. `javax.api` and `javax.transaction.api` provide classes needed by JDBC drivers and datasources, so they are added. `org.slf4j` is the well known logging facade. log4jdbc-log4j2 normally works with log4j2 as suggested by the name, but can also be configured to use slf4j which is the better option to integrate with JBoss AS. A slf4j compatible logging facade is part of JBoss AS and enables logging cofiguration as part of the container.

*Optional* dependencies for every other database driver module, that might be used together with the proxy,
have to be added for log4jdbc-log4j2 to be able to load and delegate to its classes.
Marking them as optional makes them available but allows the module to load if any of them is not available. With that it is possible to add the dependencies even if the driver is not available in some configuration.

For log4jdbc-log4j2 to use slf4j globally it has to be configured by means of a system property. To add this permanently to the JBoss configuration add the following line to the end of `bin/standalone.conf`
``` bash bin/standalone.conf
JAVA_OPTS="$JAVA_OPTS -Dlog4jdbc.spylogdelegator.name=net.sf.log4jdbc.log.slf4j.Slf4jSpyLogDelegator"
```

## Add a Datasource driver to the JBoss configuration

Before changing the JBoss configuration a backup should be created.

``` bash Copy the JBoss AS configuration to some other place
cp standalone/configuration/standalone.xml standalone/configuration/standalone.xml.org
```

Now the following changes should be done:
``` diff
--- standalone/configuration/standalone.xml.org 2013-12-05 21:10:44.357585099 +0100
+++ standalone/configuration/standalone.xml     2013-12-05 21:12:42.450172949 +0100
@@ -102,7 +102,7 @@
             <datasources>
                 <datasource jndi-name="java:jboss/datasources/ExampleDS" pool-name="ExampleDS" enabled="true" use-java-context="true">
-                    <connection-url>jdbc:h2:mem:test;DB_CLOSE_DELAY=-1</connection-url>
-                    <driver>h2</driver>
+                    <connection-url>jdbc:log4jdbc:h2:mem:test;DB_CLOSE_DELAY=-1</connection-url>
+                    <driver>log4jdbc</driver>
                     <security>
                         <user-name>sa</user-name>
                         <password>sa</password>
@@ -112,6 +112,9 @@
                     <driver name="h2" module="com.h2database.h2">
                         <xa-datasource-class>org.h2.jdbcx.JdbcDataSource</xa-datasource-class>
                     </driver>
+                    <driver name="log4jdbc" module="net.sf.log4jdbc">
+                        <datasource-class>net.sf.log4jdbc.sql.jdbcapi.DataSource</datasource-class>
+                    </driver>
                 </drivers>
             </datasources>
         </subsystem>
```

 * Replace `jdbc:h2` with `jdbc:log4jdbc:h2` in the datasource connection url. This should work with every other JDBC URL, of course. An `h2` datasource is part of the JBoss standard configuration which is the reason this is used here.
 * The datasource driver needs to be replaced with `log4jdbc` which is the name the new datasource driver will be given in the next change.
    * Normale log4jdbc-log4j2 detects the real driver by the jdbc URL and delegates to it automatically. If that does not work this can be configured in the log4jdbc-log4j2 configuration manually.
 * Add a datasource driver named `log4jdbc`. This driver is not more than a registration of the datasoure class `net.sf.log4jdbc.sql.jdbcapi.DataSource` of the `net.sf.log4jdbc` module that was created before.

## Example Log Output

The following shows the log output of a JPA2 Hibernate application doing an INSERT and a SELECT having all of the `jdbc` loggers activated. Log lines from `[stdout]` are the hibernate log lines enabled by the hibernate properties `hibernate.show_sql` and `hibernate.format_sql`.

```
14:20:11,261 INFO  [jdbc.connection] (ServerService Thread Pool -- 48) 1. Connection opened
14:20:11,261 INFO  [jdbc.audit] (ServerService Thread Pool -- 48) 1. Connection.new Connection returned·

[..]

14:22:55,621 INFO  [stdout] (http-/127.0.0.1:8080-1) Hibernate: 
14:22:55,621 INFO  [stdout] (http-/127.0.0.1:8080-1)     select
14:22:55,621 INFO  [stdout] (http-/127.0.0.1:8080-1)         nextval ('hibernate_sequence')
14:22:55,622 INFO  [jdbc.audit] (http-/127.0.0.1:8080-1) 1. PreparedStatement.new PreparedStatement returned 
14:22:55,622 INFO  [jdbc.audit] (http-/127.0.0.1:8080-1) 1. Connection.prepareStatement(select nextval ('hibernate_sequence'), 1003, 1007) returned net.sf.log4jdbc.sql.jdbcapi.PreparedStatementSpy@4c0a9658
14:22:55,623 INFO  [jdbc.sqlonly] (http-/127.0.0.1:8080-1) select nextval ('hibernate_sequence') 

14:22:55,658 INFO  [jdbc.sqltiming] (http-/127.0.0.1:8080-1) select nextval ('hibernate_sequence') 
 {executed in 35 msec}
14:22:55,658 INFO  [jdbc.resultset] (http-/127.0.0.1:8080-1) 1. ResultSet.new ResultSet returned 
14:22:55,658 INFO  [jdbc.audit] (http-/127.0.0.1:8080-1) 1. PreparedStatement.executeQuery() returned net.sf.log4jdbc.sql.jdbcapi.ResultSetSpy@3398105a
14:22:55,659 INFO  [jdbc.resultset] (http-/127.0.0.1:8080-1) 1. ResultSet.next() returned true
14:22:55,662 INFO  [jdbc.resultset] (http-/127.0.0.1:8080-1) 1. ResultSet.getLong(1) returned 1
14:22:55,663 INFO  [jdbc.resultset] (http-/127.0.0.1:8080-1) 1. ResultSet.wasNull() returned false
14:22:55,663 INFO  [jdbc.resultsettable] (http-/127.0.0.1:8080-1) 
|--------|
|nextval |
|--------|
|1       |
|--------|

14:22:55,664 INFO  [jdbc.resultset] (http-/127.0.0.1:8080-1) 1. ResultSet.close() returned void
14:22:55,664 INFO  [jdbc.audit] (http-/127.0.0.1:8080-1) 1. PreparedStatement.getMaxRows() returned 0
14:22:55,664 INFO  [jdbc.audit] (http-/127.0.0.1:8080-1) 1. PreparedStatement.getQueryTimeout() returned 0
14:22:55,664 INFO  [jdbc.audit] (http-/127.0.0.1:8080-1) 1. PreparedStatement.close() returned 
14:22:55,664 INFO  [jdbc.audit] (http-/127.0.0.1:8080-1) 1. Connection.getWarnings() returned null
14:22:55,665 INFO  [jdbc.audit] (http-/127.0.0.1:8080-1) 1. Connection.clearWarnings() returned 
14:22:55,720 INFO  [stdout] (http-/127.0.0.1:8080-1) Hibernate: 
14:22:55,721 INFO  [stdout] (http-/127.0.0.1:8080-1)     insert 
14:22:55,721 INFO  [stdout] (http-/127.0.0.1:8080-1)     into
14:22:55,721 INFO  [stdout] (http-/127.0.0.1:8080-1)         Foo
14:22:55,721 INFO  [stdout] (http-/127.0.0.1:8080-1)         (a, b, version, id) 
14:22:55,722 INFO  [stdout] (http-/127.0.0.1:8080-1)     values
14:22:55,722 INFO  [stdout] (http-/127.0.0.1:8080-1)         (?, ?, ?, ?)
14:22:55,722 INFO  [jdbc.audit] (http-/127.0.0.1:8080-1) 1. PreparedStatement.new PreparedStatement returned 
14:22:55,722 INFO  [jdbc.audit] (http-/127.0.0.1:8080-1) 1. Connection.prepareStatement(insert into Foo (a, b, version, id) values (?, ?, ?, ?), 1003, 1007) returned net.sf.log4jdbc.sql.jdbcapi.PreparedStatementSpy@46443961
14:22:55,724 INFO  [jdbc.audit] (http-/127.0.0.1:8080-1) 1. PreparedStatement.setString(1, "aValue") returned 
14:22:55,725 INFO  [jdbc.audit] (http-/127.0.0.1:8080-1) 1. PreparedStatement.setString(2, "bValue") returned 
14:22:55,725 INFO  [jdbc.audit] (http-/127.0.0.1:8080-1) 1. PreparedStatement.setInt(3, 0) returned 
14:22:55,726 INFO  [jdbc.audit] (http-/127.0.0.1:8080-1) 1. PreparedStatement.setLong(4, 1) returned 
14:22:55,726 INFO  [jdbc.sqlonly] (http-/127.0.0.1:8080-1) insert into Foo (a, b, version, id) values ('aValue', 'bValue', 0, 1) 

14:22:55,727 INFO  [jdbc.sqltiming] (http-/127.0.0.1:8080-1) insert into Foo (a, b, version, id) values ('aValue', 'bValue', 0, 1) 
 {executed in 1 msec}
14:22:55,728 INFO  [jdbc.audit] (http-/127.0.0.1:8080-1) 1. PreparedStatement.executeUpdate() returned 1
14:22:55,728 INFO  [jdbc.audit] (http-/127.0.0.1:8080-1) 1. PreparedStatement.getMaxRows() returned 0
14:22:55,728 INFO  [jdbc.audit] (http-/127.0.0.1:8080-1) 1. PreparedStatement.getQueryTimeout() returned 0
14:22:55,728 INFO  [jdbc.audit] (http-/127.0.0.1:8080-1) 1. PreparedStatement.close() returned 
14:22:55,729 INFO  [jdbc.audit] (http-/127.0.0.1:8080-1) 1. Connection.getWarnings() returned null
14:22:55,730 INFO  [jdbc.audit] (http-/127.0.0.1:8080-1) 1. Connection.clearWarnings() returned 
14:22:55,761 INFO  [jdbc.audit] (http-/127.0.0.1:8080-1) 1. Connection.commit() returned 
14:23:05,089 INFO  [stdout] (http-/127.0.0.1:8080-1) Hibernate: 
14:23:05,089 INFO  [stdout] (http-/127.0.0.1:8080-1)     select
14:23:05,090 INFO  [stdout] (http-/127.0.0.1:8080-1)         distinct foo0_.id as id0_,
14:23:05,090 INFO  [stdout] (http-/127.0.0.1:8080-1)         foo0_.a as a0_,
14:23:05,090 INFO  [stdout] (http-/127.0.0.1:8080-1)         foo0_.b as b0_,
14:23:05,090 INFO  [stdout] (http-/127.0.0.1:8080-1)         foo0_.version as version0_ 
14:23:05,090 INFO  [stdout] (http-/127.0.0.1:8080-1)     from
14:23:05,091 INFO  [stdout] (http-/127.0.0.1:8080-1)         Foo foo0_ 
14:23:05,091 INFO  [stdout] (http-/127.0.0.1:8080-1)     order by
14:23:05,091 INFO  [stdout] (http-/127.0.0.1:8080-1)         foo0_.id
14:23:05,092 INFO  [jdbc.audit] (http-/127.0.0.1:8080-1) 1. PreparedStatement.new PreparedStatement returned 
14:23:05,092 INFO  [jdbc.audit] (http-/127.0.0.1:8080-1) 1. Connection.prepareStatement(select distinct foo0_.id as id0_, foo0_.a as a0_, foo0_.b as b0_, foo0_.version as version0_ from Foo foo0_ order by foo0_.id, 1003, 1007) returned net.sf.log4jdbc.sql.jdbcapi.PreparedStatementSpy@1735e74d
14:23:05,093 INFO  [jdbc.sqlonly] (http-/127.0.0.1:8080-1) select distinct foo0_.id as id0_, foo0_.a as a0_, foo0_.b as b0_, foo0_.version as version0_ 
from Foo foo0_ order by foo0_.id 

14:23:05,094 INFO  [jdbc.sqltiming] (http-/127.0.0.1:8080-1) select distinct foo0_.id as id0_, foo0_.a as a0_, foo0_.b as b0_, foo0_.version as version0_ 
from Foo foo0_ order by foo0_.id 
 {executed in 1 msec}
14:23:05,094 INFO  [jdbc.resultset] (http-/127.0.0.1:8080-1) 1. ResultSet.new ResultSet returned 
14:23:05,094 INFO  [jdbc.audit] (http-/127.0.0.1:8080-1) 1. PreparedStatement.executeQuery() returned net.sf.log4jdbc.sql.jdbcapi.ResultSetSpy@3d91c735
14:23:05,095 INFO  [jdbc.resultset] (http-/127.0.0.1:8080-1) 1. ResultSet.next() returned true
14:23:05,096 INFO  [jdbc.resultset] (http-/127.0.0.1:8080-1) 1. ResultSet.getLong(id0_) returned 1
14:23:05,096 INFO  [jdbc.resultset] (http-/127.0.0.1:8080-1) 1. ResultSet.wasNull() returned false
14:23:05,099 INFO  [jdbc.resultset] (http-/127.0.0.1:8080-1) 1. ResultSet.getString(a0_) returned aValue
14:23:05,099 INFO  [jdbc.resultset] (http-/127.0.0.1:8080-1) 1. ResultSet.wasNull() returned false
14:23:05,099 INFO  [jdbc.resultset] (http-/127.0.0.1:8080-1) 1. ResultSet.getString(b0_) returned bValue
14:23:05,100 INFO  [jdbc.resultset] (http-/127.0.0.1:8080-1) 1. ResultSet.wasNull() returned false
14:23:05,100 INFO  [jdbc.resultset] (http-/127.0.0.1:8080-1) 1. ResultSet.getInt(version0_) returned 0
14:23:05,100 INFO  [jdbc.resultset] (http-/127.0.0.1:8080-1) 1. ResultSet.wasNull() returned false
14:23:05,102 INFO  [jdbc.resultsettable] (http-/127.0.0.1:8080-1) 
|-----|-------|-------|----------|
|id0_ |a0_    |b0_    |version0_ |
|-----|-------|-------|----------|
|1    |aValue |bValue |0         |
|-----|-------|-------|----------|

14:23:05,103 INFO  [jdbc.resultset] (http-/127.0.0.1:8080-1) 1. ResultSet.next() returned false
14:23:05,103 INFO  [jdbc.resultset] (http-/127.0.0.1:8080-1) 1. ResultSet.close() returned void
14:23:05,104 INFO  [jdbc.audit] (http-/127.0.0.1:8080-1) 1. PreparedStatement.getMaxRows() returned 0
14:23:05,104 INFO  [jdbc.audit] (http-/127.0.0.1:8080-1) 1. PreparedStatement.getQueryTimeout() returned 0
14:23:05,104 INFO  [jdbc.audit] (http-/127.0.0.1:8080-1) 1. PreparedStatement.close() returned 
14:23:05,105 INFO  [jdbc.audit] (http-/127.0.0.1:8080-1) 1. Connection.getWarnings() returned null
14:23:05,105 INFO  [jdbc.audit] (http-/127.0.0.1:8080-1) 1. Connection.clearWarnings() returned 
14:23:05,106 INFO  [jdbc.audit] (http-/127.0.0.1:8080-1) 1. Connection.commit() returned 

[..]

15:04:19,298 INFO  [jdbc.audit] (IdleRemover) 1. Connection.rollback() returned 
15:04:19,301 INFO  [jdbc.connection] (IdleRemover) 1. Connection closed 
15:04:19,302 INFO  [jdbc.audit] (IdleRemover) 1. Connection.close() returned 
```

## Adjust the logging configuration
Since the `jdbc` loggers provide overlapping information the logging should be configured to match ones needs.
In the following log4jdbc-log4j2 logging is restricted to `jdbc.timing` output. For details of the logging configuration see the log4jdbc-log4j2 [documentation](https://code.google.com/p/log4jdbc-log4j2/) and especially the section for the slf4j loggers like `jdbc.sqlonly`, `jdbc.sqltiming` and so forth.

Add the following lines next to the other logger elements in the JBoss configuration:
``` xml standalone/configuration/standalone.xml
            <logger category="jdbc">
                <level name="FATAL"/>
            </logger>
            <logger category="jdbc.sqltiming">
                <level name="INFO"/>
            </logger>
```
Of course you can use the [JBoss CLI](https://docs.jboss.org/author/display/AS72/Admin+Guide#AdminGuide-CommandLineInterface) if you like.

To turn of the JDBC tracing completely and remove the overhead it is enough to set jdbc logging completely to `FATAL` by removing the second logger element added before, no need to undo all the rest of the configuration made above.


