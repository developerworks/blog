## 第一步: 创建JOOQ的配置文件

在 **src/main/resources** 目录中创建 **jooq.xml** 配置文件, 内容如下:

```
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<configuration>
  <!-- Configure the database connection here -->
  <jdbc>
    <driver>com.mysql.jdbc.Driver</driver>
    <url>jdbc:mysql://localhost:3306/testdb?useUnicode=true&amp;characterEncoding=UTF-8</url>
    <user>root</user>
    <password>root</password>
    <!-- You can also pass user/password and other JDBC properties in the optional properties tag: -->
    <!-- <properties>
      <property>
        <key>user</key>
        <value>root</value>
      </property>
      <property>
        <key>password</key>
        <value>root</value>
      </property>
    </properties> -->
  </jdbc>
  <generator>
    <database>
      <!-- The database dialect from jooq-meta. Available dialects are
           named org.util.[database].[database]Database.

           Natively supported values are:

               org.jooq.util.ase.ASEDatabase
               org.jooq.util.cubrid.CUBRIDDatabase
               org.jooq.util.db2.DB2Database
               org.jooq.util.derby.DerbyDatabase
               org.jooq.util.firebird.FirebirdDatabase
               org.jooq.util.h2.H2Database
               org.jooq.util.hsqldb.HSQLDBDatabase
               org.jooq.util.informix.InformixDatabase
               org.jooq.util.ingres.IngresDatabase
               org.jooq.util.mariadb.MariaDBDatabase
               org.jooq.util.mysql.MySQLDatabase
               org.jooq.util.oracle.OracleDatabase
               org.jooq.util.postgres.PostgresDatabase
               org.jooq.util.sqlite.SQLiteDatabase
               org.jooq.util.sqlserver.SQLServerDatabase
               org.jooq.util.sybase.SybaseDatabase

           This value can be used to reverse-engineer generic JDBC DatabaseMetaData (e.g. for MS Access)

               org.jooq.util.jdbc.JDBCDatabase

           This value can be used to reverse-engineer standard jOOQ-meta XML formats

               org.jooq.util.xml.XMLDatabase

           You can also provide your own org.jooq.util.Database implementation
           here, if your database is currently not supported -->
      <name>org.jooq.util.mysql.MySQLDatabase</name>
      <!-- All elements that are generated from your schema (A Java regular expression.
           Use the pipe to separate several expressions) Watch out for
           case-sensitivity. Depending on your database, this might be
           important!

           You can create case-insensitive regular expressions using this syntax: (?i:expr)

           Whitespace is ignored and comments are possible.
           -->
      <includes>.*</includes>
      <!-- All elements that are excluded from your schema (A Java regular expression.
           Use the pipe to separate several expressions). Excludes match before
           includes, i.e. excludes have a higher priority -->
      <excludes></excludes>
      <!-- The schema that is used locally as a source for meta information.
           This could be your development schema or the production schema, etc
           This cannot be combined with the schemata element.

           If left empty, jOOQ will generate all available schemata. See the
           manual's next section to learn how to generate several schemata -->
      <inputSchema>testdb</inputSchema>
    </database>
    <generate>
      <!-- Generation flags: See advanced configuration properties -->
    </generate>
    <target>
      <!-- The destination package of your generated classes (within the
           destination directory)

           jOOQ may append the schema name to this package if generating multiple schemas,
           e.g. org.jooq.your.packagename.schema1
                org.jooq.your.packagename.schema2 -->
      <packageName>jooq.generated</packageName>
      <!-- The destination directory of your generated classes -->
      <directory>src/main/java</directory>
    </target>
  </generator>
</configuration>
```

## 第二步: 配置 pom.xml

配置文件路径定义

```
<properties>
    ...
    <jooq.configuration.file>${project.basedir}/src/main/resources/config/jooq.xml</jooq.configuration.file>
</properties>
```

插件配置

```
<plugin>
  <!-- Specify the maven code generator plugin -->
  <!-- Use org.jooq            for the Open Source Edition
      org.jooq.pro        for commercial editions,
      org.jooq.pro-java-6 for commercial editions with Java 6 support,
      org.jooq.trial      for the free trial edition

  Note: Only the Open Source Edition is hosted on Maven Central.
        Import the others manually from your distribution -->
  <groupId>org.jooq</groupId>
  <artifactId>jooq-codegen-maven</artifactId>
  <version>3.10.1</version>
  <executions>
    <execution>
      <id>jooq-codegen</id>
      <phase>generate-sources</phase>
      <goals>
        <goal>generate</goal>
      </goals>
      <configuration>
        <skip>${jooq.skip.generation}</skip>
        <configurationFile>${jooq.configuration.file}</configurationFile>
      </configuration>
    </execution>
  </executions>
</plugin>
```

## 第三步

```
mvn compile
```

编译完成如果没有错误, 在 **src/main/java** 中会生成 **jooq.generated** 包:


![clipboard.png](https://segmentfault.com/img/bVXR4C)




