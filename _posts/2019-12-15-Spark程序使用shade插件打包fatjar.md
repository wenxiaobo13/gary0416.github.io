---
layout:     post
title:      Spark程序使用shade插件打包fatjar
subtitle:   
date:       2019-12-15
author:     gary
header-img: 
catalog: true
tags:
    - Spark
---

# 背景
Spark On Yarn时，运行环境已有一些jar，所以打包时可排除掉以减小体积和避免冲突。

以HDP-2.6.3.0-235为例，在yarn的container日志中，可以看到在启动java进程时的CLASSPATH定义：
```
export CLASSPATH="$PWD:$PWD/__spark_conf__:$PWD/__spark_libs__/*:/usr/hdp/2.6.3.0-235/hadoop/conf:/usr/hdp/2.6.3.0-235/hadoop/*:/usr/hdp/2.6.3.0-235/hadoop/lib/*:/usr/hdp/current/hadoop-hdfs-client/*:/usr/hdp/current/hadoop-hdfs-client/lib/*:/usr/hdp/current/hadoop-yarn-client/*:/usr/hdp/current/hadoop-yarn-client/lib/*:/usr/hdp/current/ext/hadoop/*:$PWD/mr-framework/hadoop/share/hadoop/mapreduce/*:$PWD/mr-framework/hadoop/share/hadoop/mapreduce/lib/*:$PWD/mr-framework/hadoop/share/hadoop/common/*:$PWD/mr-framework/hadoop/share/hadoop/common/lib/*:$PWD/mr-framework/hadoop/share/hadoop/yarn/*:$PWD/mr-framework/hadoop/share/hadoop/yarn/lib/*:$PWD/mr-framework/hadoop/share/hadoop/hdfs/*:$PWD/mr-framework/hadoop/share/hadoop/hdfs/lib/*:$PWD/mr-framework/hadoop/share/hadoop/tools/lib/*:/usr/hdp/2.6.3.0-235/hadoop/lib/hadoop-lzo-0.6.0.2.6.3.0-235.jar:/etc/hadoop/conf/secure:/usr/hdp/current/ext/hadoop/*"
```
其中$PWD的当前目录已包含了我们spark-submit提交上去的jar和spark2-hdp-yarn-archive.tar.gz。

结论：
1. 根据上面的信息，可得知大致优先级，但同目录下的jar的加载顺序就看心情了，所以尽量避免，别乱依赖jar，遇错时往这看。
2. 遇到jar版本冲突时，要么打包时relocate，要么确认自己的优先级确实更高。

关于jar的加载顺序，可以参考文档https://docs.oracle.com/javase/7/docs/technotes/tools/solaris/classpath.html，提到：The order in which the JAR files in a directory are enumerated in the expanded class path is not specified and may vary from platform to platform and even from moment to moment on the same machine。感兴趣的可以进一步往这看http://hg.openjdk.java.net/jdk8/jdk8/hotspot/file/87ee5ee27509/src/os/linux/vm/。
 
解决：
1. maven pom中用scope可以从根本上解决，但对开发人员有较高要求，需要精确的引入和排除大量依赖。
2. 平台组建议开发组引入parent项目来排除这些jar(maven plugin无法import，只有依赖能导入，好在spark程序不像spring boot那样需要引入parent)。

# spark parent
以下供参考，通过写程序解析jar包里的pom+自动查询maven中央库，最后手动修正得出，在batch/streaming/访问hive和ES等场景测试通过。
```
<?xml version="1.0" encoding="UTF-8"?>

<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.gary.common.spark</groupId>
    <artifactId>hdp-spark-parent</artifactId>
    <version>0.0.1</version>
    <packaging>pom</packaging>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <maven.compiler.source>1.8</maven.compiler.source>
        <maven.compiler.target>1.8</maven.compiler.target>
        <spark.version>2.2.0.2.6.3.0-235</spark.version>
        <scala.binary.version>2.11</scala.binary.version>
        <spark-hive.version>1.21.2.2.6.3.0-235</spark-hive.version>
        <pmml-evaluator.version>1.4.14</pmml-evaluator.version>
        <jpmml-sparkml.version>1.3.14</jpmml-sparkml.version>
        <maven-shade-plugin.version>3.2.1</maven-shade-plugin.version>
        <shade.mainClass></shade.mainClass>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-core_${scala.binary.version}</artifactId>
            <version>${spark.version}</version>
        </dependency>
    </dependencies>
    
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.apache.spark</groupId>
                <artifactId>spark-streaming_${scala.binary.version}</artifactId>
                <version>${spark.version}</version>
            </dependency>
            <dependency>
                <groupId>org.apache.spark</groupId>
                <artifactId>spark-streaming-kafka-0-10_${scala.binary.version}</artifactId>
                <version>${spark.version}</version>
            </dependency>
            <dependency>
                <groupId>org.apache.spark</groupId>
                <artifactId>spark-sql_${scala.binary.version}</artifactId>
                <version>${spark.version}</version>
            </dependency>
            <dependency>
                <groupId>org.apache.spark</groupId>
                <artifactId>spark-hive_${scala.binary.version}</artifactId>
                <version>${spark.version}</version>
            </dependency>
            <dependency>
                <groupId>org.spark-project.hive</groupId>
                <artifactId>hive-jdbc</artifactId>
                <version>${spark-hive.version}</version>
            </dependency>
            <dependency>
                <groupId>org.apache.spark</groupId>
                <artifactId>spark-mllib_${scala.binary.version}</artifactId>
                <version>${spark.version}</version>
                <exclusions>
                    <exclusion>
                        <groupId>org.jpmml</groupId>
                        <artifactId>pmml-model</artifactId>
                    </exclusion>
                </exclusions>
            </dependency>
            <dependency>
                <groupId>org.apache.spark</groupId>
                <artifactId>spark-graphx_${scala.binary.version}</artifactId>
                <version>${spark.version}</version>
            </dependency>
            <dependency>
                <groupId>org.jpmml</groupId>
                <artifactId>jpmml-sparkml</artifactId>
                <version>${jpmml-sparkml.version}</version>
                <exclusions>
                    <exclusion>
                        <artifactId>guava</artifactId>
                        <groupId>com.google.guava</groupId>
                    </exclusion>
                </exclusions>
            </dependency>
            <dependency>
                <groupId>org.jpmml</groupId>
                <artifactId>pmml-evaluator</artifactId>
                <version>${pmml-evaluator.version}</version>
            </dependency>
            <dependency>
                <groupId>org.jpmml</groupId>
                <artifactId>pmml-evaluator-extension</artifactId>
                <version>${pmml-evaluator.version}</version>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <distributionManagement>
        <repository>
            <id>releases</id>
            <url>http://maven.gary.com/nexus/content/repositories/devops/</url>
        </repository>
    </distributionManagement>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.8.1</version>
                <configuration>
                    <source>${maven.compiler.source}</source>
                    <target>${maven.compiler.target}</target>
                    <encoding>${project.build.sourceEncoding}</encoding>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>2.22.2</version>
                <configuration>
                    <skip>true</skip>
                </configuration>
            </plugin>
            <plugin>
                <groupId>net.alchim31.maven</groupId>
                <artifactId>scala-maven-plugin</artifactId>
                <version>4.3.0</version>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-source-plugin</artifactId>
                <version>3.2.0</version>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-shade-plugin</artifactId>
                <version>${maven-shade-plugin.version}</version>
                <configuration>
                    <artifactSet>
                        <excludes>
                            <exclude>antlr:antlr</exclude>
                            <exclude>aopalliance:aopalliance</exclude>
                            <exclude>asm:asm</exclude>
                            <exclude>com.amazonaws:aws-java-sdk-core</exclude>
                            <exclude>com.amazonaws:aws-java-sdk-kms</exclude>
                            <exclude>com.amazonaws:aws-java-sdk-s3</exclude>
                            <exclude>com.chuusai:shapeless_2.11</exclude>
                            <exclude>com.clearspring.analytics:stream</exclude>
                            <exclude>com.codahale.metrics:metrics-core</exclude>
                            <exclude>com.esotericsoftware:kryo-shaded</exclude>
                            <exclude>com.esotericsoftware:minlog</exclude>
                            <exclude>com.fasterxml.jackson.core:jackson-annotations</exclude>
                            <exclude>com.fasterxml.jackson.core:jackson-core</exclude>
                            <exclude>com.fasterxml.jackson.core:jackson-databind</exclude>
                            <exclude>com.fasterxml.jackson.dataformat:jackson-dataformat-cbor</exclude>
                            <exclude>com.fasterxml.jackson.module:jackson-module-paranamer</exclude>
                            <exclude>com.fasterxml.jackson.module:jackson-module-scala_2.11</exclude>
                            <exclude>com.github.fommil.netlib:core</exclude>
                            <exclude>com.github.rwl:jtransforms</exclude>
                            <exclude>com.google.code.findbugs:jsr305</exclude>
                            <exclude>com.google.code.gson:gson</exclude>
                            <exclude>com.google.inject.extensions:guice-servlet</exclude>
                            <exclude>com.google.inject:guice</exclude>
                            <exclude>com.googlecode.javaewah:JavaEWAH</exclude>
                            <exclude>com.jamesmurty.utils:java-xmlbuilder</exclude>
                            <exclude>com.jcraft:jsch</exclude>
                            <exclude>com.jolbox:bonecp</exclude>
                            <exclude>com.microsoft.azure:azure-data-lake-store-sdk</exclude>
                            <exclude>com.microsoft.azure:azure-keyvault-core</exclude>
                            <exclude>com.microsoft.azure:azure-storage</exclude>
                            <exclude>com.nimbusds:nimbus-jose-jwt</exclude>
                            <exclude>com.ning:compress-lzf</exclude>
                            <exclude>com.squareup.okhttp:okhttp</exclude>
                            <exclude>com.squareup.okio:okio</exclude>
                            <exclude>com.sun.jersey.contribs:jersey-guice</exclude>
                            <exclude>com.sun.jersey:jersey-client</exclude>
                            <exclude>com.sun.jersey:jersey-core</exclude>
                            <exclude>com.sun.jersey:jersey-json</exclude>
                            <exclude>com.sun.jersey:jersey-server</exclude>
                            <exclude>com.sun.xml.bind:jaxb-impl</exclude>
                            <exclude>com.thoughtworks.paranamer:paranamer</exclude>
                            <exclude>com.twitter:chill-java</exclude>
                            <exclude>com.twitter:chill_2.11</exclude>
                            <exclude>com.twitter:parquet-hadoop-bundle</exclude>
                            <exclude>com.univocity:univocity-parsers</exclude>
                            <exclude>commons-beanutils:commons-beanutils</exclude>
                            <exclude>commons-beanutils:commons-beanutils-core</exclude>
                            <exclude>commons-cli:commons-cli</exclude>
                            <exclude>commons-codec:commons-codec</exclude>
                            <exclude>commons-collections:commons-collections</exclude>
                            <exclude>commons-configuration:commons-configuration</exclude>
                            <exclude>commons-daemon:commons-daemon</exclude>
                            <exclude>commons-dbcp:commons-dbcp</exclude>
                            <exclude>commons-digester:commons-digester</exclude>
                            <exclude>commons-httpclient:commons-httpclient</exclude>
                            <exclude>commons-io:commons-io</exclude>
                            <exclude>commons-lang:commons-lang</exclude>
                            <exclude>commons-logging:commons-logging</exclude>
                            <exclude>commons-net:commons-net</exclude>
                            <exclude>commons-pool:commons-pool</exclude>
                            <exclude>de.ruedigermoeller:fst</exclude>
                            <exclude>io.airlift:aircompressor</exclude>
                            <exclude>io.dropwizard.metrics:metrics-core</exclude>
                            <exclude>io.dropwizard.metrics:metrics-graphite</exclude>
                            <exclude>io.dropwizard.metrics:metrics-json</exclude>
                            <exclude>io.dropwizard.metrics:metrics-jvm</exclude>
                            <exclude>io.netty:netty</exclude>
                            <exclude>io.netty:netty-all</exclude>
                            <exclude>javax.activation:activation</exclude>
                            <exclude>javax.annotation:javax.annotation-api</exclude>
                            <exclude>javax.inject:javax.inject</exclude>
                            <exclude>javax.jdo:jdo-api</exclude>
                            <exclude>javax.mail:mail</exclude>
                            <exclude>javax.servlet.jsp:jsp-api</exclude>
                            <exclude>javax.servlet:javax.servlet-api</exclude>
                            <exclude>javax.servlet:servlet-api</exclude>
                            <exclude>javax.transaction:jta</exclude>
                            <exclude>javax.validation:validation-api</exclude>
                            <exclude>javax.ws.rs:javax.ws.rs-api</exclude>
                            <exclude>javax.xml.bind:jaxb-api</exclude>
                            <exclude>javax.xml.stream:stax-api</exclude>
                            <exclude>javolution:javolution</exclude>
                            <exclude>jline:jline</exclude>
                            <exclude>joda-time:joda-time</exclude>
                            <exclude>junit:junit</exclude>
                            <exclude>log4j:apache-log4j-extras</exclude>
                            <exclude>log4j:log4j</exclude>
                            <exclude>mx4j:mx4j</exclude>
                            <exclude>net.hydromatic:eigenbase-properties</exclude>
                            <exclude>net.iharder:base64</exclude>
                            <exclude>net.java.dev.jets3t:jets3t</exclude>
                            <exclude>net.jcip:jcip-annotations</exclude>
                            <exclude>net.jpountz.lz4:lz4</exclude>
                            <exclude>net.minidev:json-smart</exclude>
                            <exclude>net.razorvine:pyrolite</exclude>
                            <exclude>net.sf.jpam:jpam</exclude>
                            <exclude>net.sf.opencsv:opencsv</exclude>
                            <exclude>net.sf.py4j:py4j</exclude>
                            <exclude>net.sf.supercsv:super-csv</exclude>
                            <exclude>org.antlr:ST4</exclude>
                            <exclude>org.antlr:antlr-runtime</exclude>
                            <exclude>org.antlr:antlr4-runtime</exclude>
                            <exclude>org.antlr:stringtemplate</exclude>
                            <exclude>org.apache.avro:avro</exclude>
                            <exclude>org.apache.avro:avro-ipc</exclude>
                            <exclude>org.apache.avro:avro-mapred</exclude>
                            <exclude>org.apache.calcite:calcite-avatica</exclude>
                            <exclude>org.apache.calcite:calcite-core</exclude>
                            <exclude>org.apache.calcite:calcite-linq4j</exclude>
                            <exclude>org.apache.commons:commons-compress</exclude>
                            <exclude>org.apache.commons:commons-crypto</exclude>
                            <exclude>org.apache.commons:commons-lang3</exclude>
                            <exclude>org.apache.commons:commons-math3</exclude>
                            <exclude>org.apache.curator:curator-client</exclude>
                            <exclude>org.apache.curator:curator-framework</exclude>
                            <exclude>org.apache.curator:curator-recipes</exclude>
                            <exclude>org.apache.derby:derby</exclude>
                            <exclude>org.apache.directory.api:api-asn1-api</exclude>
                            <exclude>org.apache.directory.api:api-util</exclude>
                            <exclude>org.apache.directory.server:apacheds-i18n</exclude>
                            <exclude>org.apache.directory.server:apacheds-kerberos-codec</exclude>
                            <exclude>org.apache.hadoop:hadoop-annotations</exclude>
                            <exclude>org.apache.hadoop:hadoop-auth</exclude>
                            <exclude>org.apache.hadoop:hadoop-aws</exclude>
                            <exclude>org.apache.hadoop:hadoop-azure</exclude>
                            <exclude>org.apache.hadoop:hadoop-azure-datalake</exclude>
                            <exclude>org.apache.hadoop:hadoop-client</exclude>
                            <exclude>org.apache.hadoop:hadoop-common</exclude>
                            <exclude>org.apache.hadoop:hadoop-hdfs</exclude>
                            <exclude>org.apache.hadoop:hadoop-mapreduce-client-app</exclude>
                            <exclude>org.apache.hadoop:hadoop-mapreduce-client-common</exclude>
                            <exclude>org.apache.hadoop:hadoop-mapreduce-client-core</exclude>
                            <exclude>org.apache.hadoop:hadoop-mapreduce-client-jobclient</exclude>
                            <exclude>org.apache.hadoop:hadoop-mapreduce-client-shuffle</exclude>
                            <exclude>org.apache.hadoop:hadoop-nfs</exclude>
                            <exclude>org.apache.hadoop:hadoop-openstack</exclude>
                            <exclude>org.apache.hadoop:hadoop-yarn-api</exclude>
                            <exclude>org.apache.hadoop:hadoop-yarn-applications-distributedshell</exclude>
                            <exclude>org.apache.hadoop:hadoop-yarn-applications-unmanaged-am-launcher</exclude>
                            <exclude>org.apache.hadoop:hadoop-yarn-client</exclude>
                            <exclude>org.apache.hadoop:hadoop-yarn-common</exclude>
                            <exclude>org.apache.hadoop:hadoop-yarn-registry</exclude>
                            <exclude>org.apache.hadoop:hadoop-yarn-server-applicationhistoryservice</exclude>
                            <exclude>org.apache.hadoop:hadoop-yarn-server-common</exclude>
                            <exclude>org.apache.hadoop:hadoop-yarn-server-nodemanager</exclude>
                            <exclude>org.apache.hadoop:hadoop-yarn-server-resourcemanager</exclude>
                            <exclude>org.apache.hadoop:hadoop-yarn-server-sharedcachemanager</exclude>
                            <exclude>org.apache.hadoop:hadoop-yarn-server-tests</exclude>
                            <exclude>org.apache.hadoop:hadoop-yarn-server-timeline-pluginstorage</exclude>
                            <exclude>org.apache.hadoop:hadoop-yarn-server-web-proxy</exclude>
                            <exclude>org.apache.hive:hive-exec</exclude>
                            <exclude>org.apache.htrace:htrace-core</exclude>
                            <exclude>org.apache.httpcomponents:httpclient</exclude>
                            <exclude>org.apache.httpcomponents:httpcore</exclude>
                            <exclude>org.apache.ivy:ivy</exclude>
                            <exclude>org.apache.orc:orc-core</exclude>
                            <exclude>org.apache.orc:orc-mapreduce</exclude>
                            <exclude>org.apache.parquet:parquet-column</exclude>
                            <exclude>org.apache.parquet:parquet-common</exclude>
                            <exclude>org.apache.parquet:parquet-encoding</exclude>
                            <exclude>org.apache.parquet:parquet-format</exclude>
                            <exclude>org.apache.parquet:parquet-hadoop</exclude>
                            <exclude>org.apache.parquet:parquet-jackson</exclude>
                            <exclude>org.apache.ranger:ranger-hdfs-plugin-shim</exclude>
                            <exclude>org.apache.ranger:ranger-plugin-classloader</exclude>
                            <exclude>org.apache.ranger:ranger-yarn-plugin-shim</exclude>
                            <exclude>org.apache.spark:spark-catalyst_2.11</exclude>
                            <exclude>org.apache.spark:spark-cloud_2.11</exclude>
                            <exclude>org.apache.spark:spark-core_2.11</exclude>
                            <exclude>org.apache.spark:spark-graphx_2.11</exclude>
                            <exclude>org.apache.spark:spark-hive-thriftserver_2.11</exclude>
                            <exclude>org.apache.spark:spark-hive_2.11</exclude>
                            <exclude>org.apache.spark:spark-launcher_2.11</exclude>
                            <exclude>org.apache.spark:spark-mllib-local_2.11</exclude>
                            <exclude>org.apache.spark:spark-mllib_2.11</exclude>
                            <exclude>org.apache.spark:spark-network-common_2.11</exclude>
                            <exclude>org.apache.spark:spark-network-shuffle_2.11</exclude>
                            <exclude>org.apache.spark:spark-repl_2.11</exclude>
                            <exclude>org.apache.spark:spark-sketch_2.11</exclude>
                            <exclude>org.apache.spark:spark-sql_2.11</exclude>
                            <exclude>org.apache.spark:spark-streaming_2.11</exclude>
                            <exclude>org.apache.spark:spark-tags_2.11</exclude>
                            <exclude>org.apache.spark:spark-unsafe_2.11</exclude>
                            <exclude>org.apache.spark:spark-yarn_2.11</exclude>
                            <exclude>org.apache.thrift:libfb303</exclude>
                            <exclude>org.apache.thrift:libthrift</exclude>
                            <exclude>org.apache.xbean:xbean-asm5-shaded</exclude>
                            <exclude>org.apache.zookeeper:zookeeper</exclude>
                            <exclude>org.bouncycastle:bcprov-jdk15on</exclude>
                            <exclude>org.codehaus.jackson:jackson-core-asl</exclude>
                            <exclude>org.codehaus.jackson:jackson-jaxrs</exclude>
                            <exclude>org.codehaus.jackson:jackson-mapper-asl</exclude>
                            <exclude>org.codehaus.jackson:jackson-xc</exclude>
                            <exclude>org.codehaus.janino:commons-compiler</exclude>
                            <exclude>org.codehaus.janino:janino</exclude>
                            <exclude>org.codehaus.jettison:jettison</exclude>
                            <exclude>org.datanucleus:datanucleus-api-jdo</exclude>
                            <exclude>org.datanucleus:datanucleus-core</exclude>
                            <exclude>org.datanucleus:datanucleus-rdbms</exclude>
                            <exclude>org.fusesource.leveldbjni:leveldbjni-all</exclude>
                            <exclude>org.glassfish.hk2.external:aopalliance-repackaged</exclude>
                            <exclude>org.glassfish.hk2.external:javax.inject</exclude>
                            <exclude>org.glassfish.hk2:hk2-api</exclude>
                            <exclude>org.glassfish.hk2:hk2-locator</exclude>
                            <exclude>org.glassfish.hk2:hk2-utils</exclude>
                            <exclude>org.glassfish.hk2:osgi-resource-locator</exclude>
                            <exclude>org.glassfish.jersey.bundles.repackaged:jersey-guava</exclude>
                            <exclude>org.glassfish.jersey.containers:jersey-container-servlet</exclude>
                            <exclude>org.glassfish.jersey.containers:jersey-container-servlet-core</exclude>
                            <exclude>org.glassfish.jersey.core:jersey-client</exclude>
                            <exclude>org.glassfish.jersey.core:jersey-common</exclude>
                            <exclude>org.glassfish.jersey.core:jersey-server</exclude>
                            <exclude>org.glassfish.jersey.media:jersey-media-jaxb</exclude>
                            <exclude>org.hamcrest:hamcrest-core</exclude>
                            <exclude>org.iq80.snappy:snappy</exclude>
                            <exclude>org.javassist:javassist</exclude>
                            <exclude>org.jodd:jodd-core</exclude>
                            <exclude>org.jpmml:pmml-model</exclude>
                            <exclude>org.jpmml:pmml-schema</exclude>
                            <exclude>org.json4s:json4s-ast_2.11</exclude>
                            <exclude>org.json4s:json4s-core_2.11</exclude>
                            <exclude>org.json4s:json4s-jackson_2.11</exclude>
                            <exclude>org.mockito:mockito-all</exclude>
                            <exclude>org.mortbay.jetty:jetty</exclude>
                            <exclude>org.mortbay.jetty:jetty-sslengine</exclude>
                            <exclude>org.mortbay.jetty:jetty-util</exclude>
                            <exclude>org.netlib:arpack_combined_all</exclude>
                            <exclude>org.objenesis:objenesis</exclude>
                            <exclude>org.roaringbitmap:RoaringBitmap</exclude>
                            <exclude>org.scala-lang.modules:scala-parser-combinators_2.11</exclude>
                            <exclude>org.scala-lang.modules:scala-xml_2.11</exclude>
                            <exclude>org.scala-lang:scala-compiler</exclude>
                            <exclude>org.scala-lang:scala-library</exclude>
                            <exclude>org.scala-lang:scala-reflect</exclude>
                            <exclude>org.scala-lang:scalap</exclude>
                            <exclude>org.scalanlp:breeze-macros_2.11</exclude>
                            <exclude>org.scalanlp:breeze_2.11</exclude>
                            <exclude>org.slf4j:jcl-over-slf4j</exclude>
                            <exclude>org.slf4j:jul-to-slf4j</exclude>
                            <exclude>org.slf4j:slf4j-api</exclude>
                            <exclude>org.slf4j:slf4j-log4j12</exclude>
                            <exclude>org.spark-project.hive:hive-beeline</exclude>
                            <exclude>org.spark-project.hive:hive-cli</exclude>
                            <exclude>org.spark-project.hive:hive-jdbc</exclude>
                            <exclude>org.spark-project.hive:hive-metastore</exclude>
                            <exclude>org.spark-project.spark:unused</exclude>
                            <exclude>org.spire-math:spire-macros_2.11</exclude>
                            <exclude>org.spire-math:spire_2.11</exclude>
                            <exclude>org.tukaani:xz</exclude>
                            <exclude>org.typelevel:machinist_2.11</exclude>
                            <exclude>org.typelevel:macro-compat_2.11</exclude>
                            <exclude>org.xerial.snappy:snappy-java</exclude>
                            <exclude>oro:oro</exclude>
                            <exclude>stax:stax-api</exclude>
                            <exclude>xerces:xercesImpl</exclude>
                            <exclude>xml-apis:xml-apis</exclude>
                            <exclude>xmlenc:xmlenc</exclude>
                        </excludes>
                    </artifactSet>
                    <filters>
                        <filter>
                            <artifact>*:*</artifact>
                            <excludes>
                                <exclude>META-INF/*.SF</exclude>
                                <exclude>META-INF/*.DSA</exclude>
                                <exclude>META-INF/*.RSA</exclude>
                                <exclude>hdfs-site.xml</exclude>
                                <exclude>core-site.xml</exclude>
                                <exclude>hive-site.xml</exclude>
                            </excludes>
                        </filter>
                    </filters>
                    <transformers>
                        <transformer implementation="org.apache.maven.plugins.shade.resource.AppendingTransformer">
                            <resource>META-INF/spring.handlers</resource>
                        </transformer>
                        <transformer implementation="org.apache.maven.plugins.shade.resource.AppendingTransformer">
                            <resource>META-INF/spring.schemas</resource>
                        </transformer>
                        <transformer implementation="org.apache.maven.plugins.shade.resource.ServicesResourceTransformer">
                        </transformer>
                        <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                            <mainClass>${shade.mainClass}</mainClass>
                        </transformer>
                    </transformers>
                </configuration>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals>
                            <goal>shade</goal>
                        </goals>
                    </execution>
                </executions>
                <relocations>
                    <relocation>
                        <pattern>com.google.protobuf</pattern>
                        <shadedPattern>shaded.com.google.protobuf</shadedPattern>
                    </relocation>
                    <relocation>
                        <pattern>com.google.guava</pattern>
                        <shadedPattern>shaded.com.google.guava</shadedPattern>
                    </relocation>
                    <!--
                        SPARK-15526
                        本地IDE运行时,sparkml2pmml.properties干掉
                        集群运行时,因为有shade relocate,所以需要在配置文件里指定relocate后的类名
                        src/main/resources/META-INF/sparkml2pmml.properties内容如下:
                        # Features
                        org.apache.spark.ml.feature.Binarizer = shaded.org.jpmml.sparkml.feature.BinarizerConverter
                        org.apache.spark.ml.feature.Bucketizer = shaded.org.jpmml.sparkml.feature.BucketizerConverter
                        org.apache.spark.ml.feature.ChiSqSelectorModel = shaded.org.jpmml.sparkml.feature.ChiSqSelectorModelConverter
                        org.apache.spark.ml.feature.ColumnPruner = shaded.org.jpmml.sparkml.feature.ColumnPrunerConverter
                        org.apache.spark.ml.feature.CountVectorizerModel = shaded.org.jpmml.sparkml.feature.CountVectorizerModelConverter
                        org.apache.spark.ml.feature.IDFModel = shaded.org.jpmml.sparkml.feature.IDFModelConverter
                        org.apache.spark.ml.feature.ImputerModel = shaded.org.jpmml.sparkml.feature.ImputerModelConverter
                        org.apache.spark.ml.feature.IndexToString = shaded.org.jpmml.sparkml.feature.IndexToStringConverter
                        org.apache.spark.ml.feature.Interaction = shaded.org.jpmml.sparkml.feature.InteractionConverter
                        org.apache.spark.ml.feature.MaxAbsScalerModel = shaded.org.jpmml.sparkml.feature.MaxAbsScalerModelConverter
                        org.apache.spark.ml.feature.MinMaxScalerModel = shaded.org.jpmml.sparkml.feature.MinMaxScalerModelConverter
                        org.apache.spark.ml.feature.NGram = shaded.org.jpmml.sparkml.feature.NGramConverter
                        org.apache.spark.ml.feature.OneHotEncoder = shaded.org.jpmml.sparkml.feature.OneHotEncoderConverter
                        org.apache.spark.ml.feature.PCAModel = shaded.org.jpmml.sparkml.feature.PCAModelConverter
                        org.apache.spark.ml.feature.RegexTokenizer = shaded.org.jpmml.sparkml.feature.RegexTokenizerConverter
                        org.apache.spark.ml.feature.RFormulaModel = shaded.org.jpmml.sparkml.feature.RFormulaModelConverter
                        org.apache.spark.ml.feature.SQLTransformer = shaded.org.jpmml.sparkml.feature.SQLTransformerConverter
                        org.apache.spark.ml.feature.StandardScalerModel = shaded.org.jpmml.sparkml.feature.StandardScalerModelConverter
                        org.apache.spark.ml.feature.StringIndexerModel = shaded.org.jpmml.sparkml.feature.StringIndexerModelConverter
                        org.apache.spark.ml.feature.StopWordsRemover = shaded.org.jpmml.sparkml.feature.StopWordsRemoverConverter
                        org.apache.spark.ml.feature.Tokenizer = shaded.org.jpmml.sparkml.feature.TokenizerConverter
                        org.apache.spark.ml.feature.VectorAssembler = shaded.org.jpmml.sparkml.feature.VectorAssemblerConverter
                        org.apache.spark.ml.feature.VectorAttributeRewriter = shaded.org.jpmml.sparkml.feature.VectorAttributeRewriterConverter
                        org.apache.spark.ml.feature.VectorIndexerModel = shaded.org.jpmml.sparkml.feature.VectorIndexerModelConverter
                        org.apache.spark.ml.feature.VectorSlicer = shaded.org.jpmml.sparkml.feature.VectorSlicerConverter
                        # Prediction models
                        org.apache.spark.ml.classification.DecisionTreeClassificationModel = shaded.org.jpmml.sparkml.model.DecisionTreeClassificationModelConverter
                        org.apache.spark.ml.classification.GBTClassificationModel = shaded.org.jpmml.sparkml.model.GBTClassificationModelConverter
                        org.apache.spark.ml.classification.LinearSVCModel = shaded.org.jpmml.sparkml.model.LinearSVCModelConverter
                        org.apache.spark.ml.classification.LogisticRegressionModel = shaded.org.jpmml.sparkml.model.LogisticRegressionModelConverter
                        org.apache.spark.ml.classification.MultilayerPerceptronClassificationModel = shaded.org.jpmml.sparkml.model.MultilayerPerceptronClassificationModelConverter
                        org.apache.spark.ml.classification.NaiveBayesModel = shaded.org.jpmml.sparkml.model.NaiveBayesModelConverter
                        org.apache.spark.ml.classification.RandomForestClassificationModel = shaded.org.jpmml.sparkml.model.RandomForestClassificationModelConverter
                        org.apache.spark.ml.clustering.KMeansModel = shaded.org.jpmml.sparkml.model.KMeansModelConverter
                        org.apache.spark.ml.regression.DecisionTreeRegressionModel = shaded.org.jpmml.sparkml.model.DecisionTreeRegressionModelConverter
                        org.apache.spark.ml.regression.GBTRegressionModel = shaded.org.jpmml.sparkml.model.GBTRegressionModelConverter
                        org.apache.spark.ml.regression.GeneralizedLinearRegressionModel = shaded.org.jpmml.sparkml.model.GeneralizedLinearRegressionModelConverter
                        org.apache.spark.ml.regression.LinearRegressionModel = shaded.org.jpmml.sparkml.model.LinearRegressionModelConverter
                        org.apache.spark.ml.regression.RandomForestRegressionModel = shaded.org.jpmml.sparkml.model.RandomForestRegressionModelConverter
                    -->
                    <relocation>
                        <pattern>org.dmg.pmml</pattern>
                        <shadedPattern>shaded.org.dmg.pmml</shadedPattern>
                    </relocation>
                    <relocation>
                        <pattern>org.jpmml</pattern>
                        <shadedPattern>shaded.org.jpmml</shadedPattern>
                    </relocation>
                </relocations>
            </plugin>
        </plugins>
    </build>
</project>
```

# 使用
spark程序直接：
```
<parent>
    <groupId>com.gary.common.spark</groupId>
    <artifactId>hdp-spark-parent</artifactId>
    <version>0.0.1</version>
    <relativePath/>
</parent>
```

依赖省略版本号：
```
<dependency>
    <groupId>org.apache.spark</groupId>
    <artifactId>spark-streaming_${scala.binary.version}</artifactId>
</dependency>
```

build plugin可以为空。
