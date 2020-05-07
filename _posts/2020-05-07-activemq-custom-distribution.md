---
title: ActiveMQ Custom Distribution
subtitle: How to create a custom ActiveMQ distribution
layout: post
author: F.Papon
tags: ActiveMQ docker
---

In this tutorial, I will show you how to create a custom **Apache ActiveMQ** distribution ready to deploy.
We will use the `karaf-maven-plugin` to build the distribution and the `jib-maven-plugin` to build the docker image.

The advantage to create a custom distribution is that you can have a default configuration of the ActiveMQ ready to deploy. 

## Project structure

We only need a Maven `pom.xml`.

We add a file `src/main/conf/activemq.xml` in the project to define the default configuration file of the distribution.
You can add any other ActiveMQ configuration file like logging, credentials, jetty...

```xml
<beans
  xmlns="http://www.springframework.org/schema/beans"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
  http://activemq.apache.org/schema/core http://activemq.apache.org/schema/core/activemq-core.xsd">

    <!-- Allows us to use system properties as variables in this configuration file -->
    <bean class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">
        <property name="locations">
            <value>file:${activemq.conf}/credentials.properties</value>
        </property>
    </bean>

   <!-- Allows accessing the server log -->
    <bean id="logQuery" class="io.fabric8.insight.log.log4j.Log4jLogQuery"
          lazy-init="false" scope="singleton"
          init-method="start" destroy-method="stop">
    </bean>

    <!--
        The <broker> element is used to configure the ActiveMQ broker.
    -->
    <broker xmlns="http://activemq.apache.org/schema/core" brokerName="localhost" dataDirectory="${activemq.data}">

        <destinationPolicy>
            <policyMap>
              <policyEntries>
                <policyEntry topic=">" >
                    <!-- The constantPendingMessageLimitStrategy is used to prevent
                         slow topic consumers to block producers and affect other consumers
                         by limiting the number of messages that are retained
                         For more information, see:

                         http://activemq.apache.org/slow-consumer-handling.html

                    -->
                  <pendingMessageLimitStrategy>
                    <constantPendingMessageLimitStrategy limit="1000"/>
                  </pendingMessageLimitStrategy>
                </policyEntry>
              </policyEntries>
            </policyMap>
        </destinationPolicy>


        <!--
            The managementContext is used to configure how ActiveMQ is exposed in
            JMX. By default, ActiveMQ uses the MBean server that is started by
            the JVM. For more information, see:

            http://activemq.apache.org/jmx.html
        -->
        <managementContext>
            <managementContext createConnector="false"/>
        </managementContext>

        <!--
            Configure message persistence for the broker. The default persistence
            mechanism is the KahaDB store (identified by the kahaDB tag).
            For more information, see:

            http://activemq.apache.org/persistence.html
        -->
        <persistenceAdapter>
            <kahaDB directory="${activemq.data}/custom-kahadb"/>
        </persistenceAdapter>


          <!--
            The systemUsage controls the maximum amount of space the broker will
            use before disabling caching and/or slowing down producers. For more information, see:
            http://activemq.apache.org/producer-flow-control.html
          -->
          <systemUsage>
            <systemUsage>
                <memoryUsage>
                    <memoryUsage percentOfJvmHeap="70" />
                </memoryUsage>
                <storeUsage>
                    <storeUsage limit="100 gb"/>
                </storeUsage>
                <tempUsage>
                    <tempUsage limit="50 gb"/>
                </tempUsage>
            </systemUsage>
        </systemUsage>

        <!--
            The transport connectors expose ActiveMQ over a given protocol to
            clients and other brokers. For more information, see:

            http://activemq.apache.org/configuring-transports.html
        -->
        <transportConnectors>
            <!-- DOS protection, limit concurrent connections to 1000 and frame size to 100MB -->
            <transportConnector name="openwire" uri="tcp://0.0.0.0:61616?maximumConnections=1000&amp;wireFormat.maxFrameSize=104857600"/>
            <transportConnector name="amqp" uri="amqp://0.0.0.0:5672?maximumConnections=1000&amp;wireFormat.maxFrameSize=104857600"/>
            <transportConnector name="stomp" uri="stomp://0.0.0.0:61613?maximumConnections=1000&amp;wireFormat.maxFrameSize=104857600"/>
            <transportConnector name="mqtt" uri="mqtt://0.0.0.0:1883?maximumConnections=1000&amp;wireFormat.maxFrameSize=104857600"/>
            <transportConnector name="ws" uri="ws://0.0.0.0:61614?maximumConnections=1000&amp;wireFormat.maxFrameSize=104857600"/>
        </transportConnectors>

        <!-- destroy the spring context on shutdown to stop jetty -->
        <shutdownHooks>
            <bean xmlns="http://www.springframework.org/schema/beans" class="org.apache.activemq.hooks.SpringContextHook" />
        </shutdownHooks>

    </broker>

    <!--
        Enable web consoles, REST and Ajax APIs and demos
        The web consoles requires by default login, you can disable this in the jetty.xml file

        Take a look at ${ACTIVEMQ_HOME}/conf/jetty.xml for more details
    -->
    <import resource="jetty.xml"/>

</beans>
```

### Dependencies

We are using the `maven-dependency-plugin` to download and unpack the original distribution.

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-dependency-plugin</artifactId>
    <version>3.1.1</version>
    <executions>
        <execution>
            <id>unpack-activemq</id>
            <phase>generate-resources</phase>
            <goals>
                <goal>unpack</goal>
            </goals>
            <configuration>
                <artifactItems>
                    <artifactItem>
                        <groupId>org.apache.activemq</groupId>
                        <artifactId>apache-activemq</artifactId>
                        <version>${activemq.version}</version>
                        <classifier>bin</classifier>
                        <type>tar.gz</type>
                        <outputDirectory>target/activemq</outputDirectory>
                    </artifactItem>
                </artifactItems>
            </configuration>
        </execution>
    </executions>
</plugin>
```

## Build

### Assembly

We are using the `maven-assembly-plugin` to build the assembly and a descriptor to define the modifications to apply on the distribtion.

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-assembly-plugin</artifactId>
    <version>3.1.0</version>
    <executions>
        <execution>
            <id>activemq</id>
            <phase>package</phase>
            <goals>
                <goal>single</goal>
            </goals>
            <configuration>
                <descriptors>
                    <descriptor>src/main/descriptors/activemq.xml</descriptor>
                </descriptors>
                <appendAssemblyId>false</appendAssemblyId>
                <tarLongFileMode>gnu</tarLongFileMode>
                <finalName>assembly</finalName>
            </configuration>
        </execution>
    </executions>
</plugin>
```

Descriptor in `src/main/descriptors/activemq.con`, we want to exclude some default folder like the docs, examples, webapps-demo... to make the distribution size smaller.

```xml
<assembly>
    <id>activemq</id>

    <baseDirectory>activemq</baseDirectory>

    <formats>
        <format>dir</format>
    </formats>

    <fileSets>

        <!-- bin -->
        <fileSet>
            <directory>target/classes/bin</directory>
            <outputDirectory>/bin</outputDirectory>
            <includes>
                <include>**/*</include>
            </includes>
            <lineEnding>unix</lineEnding>
            <fileMode>0755</fileMode>
            <directoryMode>755</directoryMode>
        </fileSet>

        <!-- expanded ActiveMQ -->
        <fileSet>
            <directory>target/activemq/apache-activemq-${activemq.version}</directory>
            <outputDirectory>/</outputDirectory>
            <excludes>
                <exclude>conf/activemq.xml</exclude>
                <exclude>activemq-all*.jar</exclude>
                <exclude>docs/**</exclude>
                <exclude>examples/**</exclude>
                <exclude>webapps-demo/**</exclude>
                <exclude>README.txt</exclude>
                <exclude>LICENSE</exclude>
                <exclude>NOTICE</exclude>
            </excludes>
        </fileSet>

        <!-- Copy over unix bin/* separately to get the correct file mode -->
        <fileSet>
            <directory>target/activemq/apache-activemq-${activemq.version}/bin</directory>
            <outputDirectory>/bin</outputDirectory>
            <includes>
                <include>env</include>
            </includes>
            <lineEnding>unix</lineEnding>
            <fileMode>0755</fileMode>
            <directoryMode>777</directoryMode>
        </fileSet>

        <!-- copy config files to etc -->
        <fileSet>
            <directory>src/main/conf</directory>
            <outputDirectory>/conf</outputDirectory>
            <fileMode>0644</fileMode>
            <directoryMode>777</directoryMode>
        </fileSet>
    </fileSets>

</assembly>
```

Build the distribution:

```bash
mvn clean package
```

You can see the distribution structure in `target/assembly/activemq`:

```bash
├── activemq
│   └── apache-activemq-5.15.11
│       ├── activemq-all-5.15.11.jar
│       ├── bin
│       │   ├── activemq
│       │   ├── activemq-diag
│       │   ├── activemq.jar
│       │   ├── env
│       │   ├── linux-x86-32
│       │   │   ├── activemq
│       │   │   ├── libwrapper.so
│       │   │   ├── wrapper
│       │   │   └── wrapper.conf
│       │   ├── linux-x86-64
│       │   │   ├── activemq
│       │   │   ├── libwrapper.so
│       │   │   ├── wrapper
│       │   │   └── wrapper.conf
│       │   ├── macosx
│       │   │   ├── activemq
│       │   │   ├── libwrapper.jnilib
│       │   │   ├── wrapper
│       │   │   └── wrapper.conf
│       │   └── wrapper.jar
│       ├── conf
│       │   ├── activemq.xml
│       │   ├── broker.ks
│       │   ├── broker-localhost.cert
│       │   ├── broker.ts
│       │   ├── client.ks
│       │   ├── client.ts
│       │   ├── credentials-enc.properties
│       │   ├── credentials.properties
│       │   ├── groups.properties
│       │   ├── java.security
│       │   ├── jetty-realm.properties
│       │   ├── jetty.xml
│       │   ├── jmx.access
│       │   ├── jmx.password
│       │   ├── log4j.properties
│       │   ├── logging.properties
│       │   ├── login.config
│       │   └── users.properties
│       ├── data
│       │   └── activemq.log

        ...
```

We can test the instance and check the installation by running:

```bash
sh ./target/assembly/activemq/apache-activemq-5.15.11/bin/activemq console
```

```bash
....
 Loading message broker from: xbean:activemq.xml
  INFO | Refreshing org.apache.activemq.xbean.XBeanBrokerFactory$1@4dbb42b7: startup date [Thu May 07 05:01:55 CEST 2020]; root of context hierarchy
  INFO | Using Persistence Adapter: KahaDBPersistenceAdapter[/home/blog-tutorial/activemq-docker/target/activemq/apache-activemq-5.15.11/data/kahadb]
  INFO | PListStore:[/home/blog-tutorial/activemq-docker/target/activemq/apache-activemq-5.15.11/data/localhost/tmp_storage] started
  INFO | Apache ActiveMQ 5.15.11 (localhost, ID:laptop-37883-1588820516667-0:1) is starting
  INFO | Listening for connections at: tcp://laptop:61616?maximumConnections=1000&wireFormat.maxFrameSize=104857600
  INFO | Connector openwire started
  INFO | Listening for connections at: amqp://laptop:5672?maximumConnections=1000&wireFormat.maxFrameSize=104857600
  INFO | Connector amqp started
  INFO | Listening for connections at: stomp://laptop:61613?maximumConnections=1000&wireFormat.maxFrameSize=104857600
  INFO | Connector stomp started
  INFO | Listening for connections at: mqtt://laptop:1883?maximumConnections=1000&wireFormat.maxFrameSize=104857600
  INFO | Connector mqtt started
  INFO | Starting Jetty server
  INFO | Creating Jetty connector
  WARN | ServletContext@o.e.j.s.ServletContextHandler@44de94c3{/,null,STARTING} has uncovered http methods for path: /
  INFO | Listening for connections at ws://laptop:61614?maximumConnections=1000&wireFormat.maxFrameSize=104857600
  INFO | Connector ws started
  INFO | Apache ActiveMQ 5.15.11 (localhost, ID:laptop-37883-1588820516667-0:1) started
  INFO | For help or more information please see: http://activemq.apache.org
  WARN | Store limit is 102400 mb (current store usage is 0 mb). The data directory: /home/blog-tutorial/activemq-docker/target/activemq/apache-activemq-5.15.11/data/kahadb only has 74635 mb of usable space. - resetting to maximum available disk space: 74635 mb
  INFO | ActiveMQ WebConsole available at http://0.0.0.0:8161/
  INFO | ActiveMQ Jolokia REST API available at http://0.0.0.0:8161/api/jolokia/
                                                                      
```

### JIB-Maven-Plugin

```xml
<plugin>
    <groupId>com.google.cloud.tools</groupId>
    <artifactId>jib-maven-plugin</artifactId>
    <version>1.8.0</version>
    <configuration>
        <allowInsecureRegistries>true</allowInsecureRegistries>
        <from>
            <image>openjdk:8-jre-alpine</image>
        </from>
        <to>
            <image>fpaponapache.azurecr.io/fpaponapache/activemq-custom-distrib:${project.version}</image>
            <auth>
                <username>fpaponapache</username>
                <password></password>
            </auth>
        </to>
        <container>
            <entrypoint>
                <entrypoint>/activemq/bin/activemq</entrypoint>
                <entrypoint>console</entrypoint>
            </entrypoint>
            <workingDirectory>/activemq</workingDirectory>
            <ports>
                <port>8161</port>
                <port>61616</port>
                <port>5672</port>
                <port>61613</port>
                <port>1883</port>
                <port>61614</port>
            </ports>
        </container>
        <extraDirectories>
            <paths>
                <path>target/assembly</path>
            </paths>
            <permissions>
                <permission>
                    <file>/activemq/bin/activemq</file>
                    <mode>755</mode>
                </permission>
            </permissions>
        </extraDirectories>
    </configuration>
</plugin>
```

We are using the **openjdk11 alpine jre**.

Here we are using an image name to be deployed on Azure registry.
To deploy on another registry, you can check the JIB maven plugin documentation [here](https://github.com/GoogleContainerTools/jib/tree/master/jib-maven-plugin)

We define the Karaf instance entrypoint to the  `activemq console` and expose the ports:

*  8161 for the webconsole
*  61616 for the openwire transport
*  5672 for the amqp transport
*  61613 for the stomp transport
*  1883 for the mqtt transport
*  61614 for the ws transfport

You can refer to the Apache ActiveMQ documentation to learn more about the transport configuration:

https://activemq.apache.org/configuring-transports.html

Build the image on local registry:

```
mvn jib:dockerBuild
```

NB: To build and pull on the remote registry, execute `mvn jib:build`.

You can see the image on your local docker registry:

```
docker images
>
REPOSITORY                                                  TAG                   IMAGE ID            CREATED             SIZE
fpaponapache.azurecr.io/fpaponapache/activemq-custom-distrib   1.0.0-SNAPSHOT        8ed6cc4bae93        2 days ago          136MB
```

Now you can run a container:

```
docker run -it --name activemq-custom fpaponapache.azurecr.io/fpaponapache/activemq-custom-distrib:1.0.0-SNAPSHOT
```

Find the IP of the container:

```bash
docker inspect activemq-custom

...
    "Networks": {
        "bridge": {
            "IPAMConfig": null,
            "Links": null,
            "Aliases": null,
            "NetworkID": "1fb712c7d779bcf7c7bf8af14d926eb19b676e059f04ec06a01504a79bce2e80",
            "EndpointID": "281c001dc403cad585002d2f4567a2574fcc828dd2360b518fbd746d1e39c0fa",
            "Gateway": "172.17.0.1",
            "IPAddress": "172.17.0.2",
            "IPPrefixLen": 16,
            "IPv6Gateway": "",
            "GlobalIPv6Address": "",
            "GlobalIPv6PrefixLen": 0,
            "MacAddress": "02:42:ac:11:00:02",
            "DriverOpts": null
        }
    }

```

You can access to the webconsole at http://172.17.0.2:8161/admin (user: admin / password: admin) and check the availability of the broker via openwire with the command `telnet 172.17.0.2 616161`.

## Conclusion

You now have the basic information to create an ActiveMQ custom distribution and pull it to a Docker registry.

The source of the example are available on Github [here](https://github.com/fpapon/blog-tutorial/tree/master/activemq-docker)
