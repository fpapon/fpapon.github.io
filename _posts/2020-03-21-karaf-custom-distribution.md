---
title: Karaf Custom Distribution
subtitle: How to create a custom Karaf distribution
layout: post
author: F.Papon
tags: karaf docker
---

In this tutorial, I will show you how to create a custom **Apache Karaf** distribution.
We will use the `karaf-maven-plugin` to build the distribution and the `jib-maven-plugin` to build the docker image.

One of the advantage to create a custom distribution is the pre-packaging of the default features. It's pretty convenient when 
you want to deploy in a closed network or to avoid downloading the features at the first startup of the instance. 

## Project structure

We only need a Maven `pom.xml`.

Because of security reason, the JMX ports are only available on localhost.
For the example, we want to allow JMX access outside of the docker container without binding the port to localhost, 
so We change the host to `0.0.0.0` in the `etc/org.apache.karaf.management.cfg` config file.

We add a file `src/main/karaf/assembly-property-edits.xml` in the project.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<property-edits xmlns="http://karaf.apache.org/tools/property-edits/1.0.0">
    <edits>
        <edit>
            <file>org.apache.karaf.management.cfg</file>
            <operation>put</operation>
            <key>rmiRegistryHost</key>
            <value>0.0.0.0</value>
        </edit>
        <edit>
            <file>org.apache.karaf.management.cfg</file>
            <operation>put</operation>
            <key>rmiServerHost</key>
            <value>0.0.0.0</value>
        </edit>
    </edits>
</property-edits>
```

### Packaging

According to the `karaf-maven-plugin` we use the package type:

```xml
<packaging>karaf-assembly</packaging>
```

### Dependencies

```xml
<dependencies>
    <dependency>
        <groupId>org.apache.karaf.features</groupId>
        <artifactId>framework</artifactId>
        <version>${karaf.runtime.version}</version>
        <type>kar</type>
    </dependency>
    <dependency>
        <groupId>org.apache.karaf.features</groupId>
        <artifactId>framework</artifactId>
        <version>${karaf.runtime.version}</version>
        <classifier>features</classifier>
        <type>xml</type>
        <scope>runtime</scope>
    </dependency>
    <dependency>
        <groupId>org.apache.karaf.features</groupId>
        <artifactId>standard</artifactId>
        <version>${karaf.runtime.version}</version>
        <classifier>features</classifier>
        <type>xml</type>
    </dependency>
    <dependency>
        <groupId>org.apache.karaf.decanter</groupId>
        <artifactId>apache-karaf-decanter</artifactId>
        <version>${decanter.version}</version>
        <classifier>features</classifier>
        <type>xml</type>
        <scope>runtime</scope>
    </dependency>
    <dependency>
        <groupId>org.slf4j</groupId>
        <artifactId>slf4j-api</artifactId>
        <version>${slf4j.version}</version>
        <scope>provided</scope>
    </dependency>
</dependencies>
```

As a minimum runtime, we have to add the Karaf `framework` and `standard` features.
For the example, we add the Karaf Decanter feature repository.

## Build

### Karaf-Maven-Plugin

```xml
<plugin>
    <groupId>org.apache.karaf.tooling</groupId>
    <artifactId>karaf-maven-plugin</artifactId>
    <extensions>true</extensions>
    <configuration>
        <finalName>${project.artifactId}</finalName>
        <bootFeatures>
            <feature>wrap</feature>
            <feature>bundle</feature>
            <feature>config</feature>
            <feature>system</feature>
            <feature>feature</feature>
            <feature>package</feature>
            <feature>log</feature>
            <feature>ssh</feature>
            <feature>instance</feature>
            <feature>shell</feature>
            <feature>management</feature>
            <feature>service</feature>
            <feature>jaas</feature>
            <feature>deployer</feature>
            <feature>diagnostic</feature>
            <feature>scr</feature>
            <feature>http</feature>
            <feature>war</feature>
            <feature>decanter-collector-rest-servlet</feature>
        </bootFeatures>
        <installedFeatures>
            <feature>aries-blueprint</feature>
            <feature>shell-compat</feature>
        </installedFeatures>
        <startupFeatures>
            <feature>eventadmin</feature>
        </startupFeatures>
        <archiveTarGz>false</archiveTarGz>
        <archiveZip>false</archiveZip>
        <workDirectory>${project.build.directory}/assembly/karaf</workDirectory>
        <pathPrefix>karaf</pathPrefix>
        <writeProfiles>true</writeProfiles>
    </configuration>
</plugin>
```

Define the **feature** list we want to include in the distribution:

*  `<startupFeatures>`: features loaded during the startup of the runtime.
*  `<bootFeatures>`: features installed and started after the startup of the runtime.
*  `<installedFeatures>`: features included in the distribution but not started.

We activate the profile output to check the feature configuration, they will be generated in `$KARAF_HOME/etc/profiles` folder:

```xml
<writeProfiles>true</writeProfiles>
```

As we want to use the distribution in **Docker**, we deactivate the `<archiveZip>` and `<archiveTarGz>` options:

```xml
<archiveZip>false</archiveZip>
```

Define the output directory:

```xml
<workDirectory>${project.build.directory}/assembly/karaf</workDirectory>
```

For the example, we want to install and start the Karaf Decanter rest collector, we add it in the `bootFeature` list:

```xml
<feature>decanter-collector-rest-servlet</feature>
```

Build the distribution:

```bash
mvn clean package
```

You can see the distribution structure in `target/assembly/karaf`:

```bash
└── karaf
    ├── bin
    │   └── contrib
    ├── data
    │   └── tmp
    ├── deploy
    ├── etc
    │   ├── profiles
    │   │   └── generated
    │   │       ├── boot.profile
    │   │       ├── installed.profile
    │   │       └── startup.profile
    │   └── scripts
    ├── lib
    │   ├── boot
    │   ├── endorsed
    │   ├── ext
    │   └── jdk9plus
    ├── META-INF
    └── system
        ├── javax
        ...
```

Check the `etc/org.apache.karaf.management.cfg` management config file:

```properties
#Modified by org.apache.karaf.tools.utils.KarafPropertiesFile
#Sat Mar 21 18:11:12 CET 2020
daemon=true
jmxRealm=karaf
jmxmpEnabled=false
jmxmpHost=127.0.0.1
jmxmpObjectName=connector\:name\=jmxmp
jmxmpPort=9999
jmxmpServiceUrl=service\:jmx\:jmxmp\://${jmxmpHost}\:${jmxmpPort}
objectName=connector\:name\=rmi
rmiRegistryHost=0.0.0.0
rmiRegistryPort=1099
rmiServerHost=0.0.0.0
rmiServerPort=44444
serviceUrl=service\:jmx\:rmi\://${rmiServerHost}\:${rmiServerPort}/jndi/rmi\://${rmiRegistryHost}\:${rmiRegistryPort}/karaf-${karaf.name}
threaded=true
```

We can test the instance and check the installation of the Karaf Decanter **rest collector**:

```bash
sh ./target/assembly/karaf/bin/karaf
```

```bash
        __ __                  ____      
       / //_/____ __________ _/ __/      
      / ,<  / __ `/ ___/ __ `/ /_        
     / /| |/ /_/ / /  / /_/ / __/        
    /_/ |_|\__,_/_/   \__,_/_/         

  Apache Karaf (4.3.0.RC1)

Hit '<tab>' for a list of available commands
and '[cmd] --help' for help on a specific command.
Hit '<ctrl-d>' or type 'system:shutdown' or 'logout' to shutdown Karaf.

karaf@root()> list
START LEVEL 100 , List Threshold: 50
 ID │ State  │ Lvl │ Version   │ Name
────┼────────┼─────┼───────────┼────────────────────────────────────────────────────────────
 27 │ Active │  80 │ 2.2.0     │ Apache Karaf :: Decanter :: API
 28 │ Active │  80 │ 2.2.0     │ Apache Karaf :: Decanter :: Collector :: REST :: Servlet
 29 │ Active │  80 │ 2.2.0     │ Apache Karaf :: Decanter :: Marshaller :: CSV
 30 │ Active │  80 │ 2.2.0     │ Apache Karaf :: Decanter :: Marshaller :: Json
 31 │ Active │  80 │ 2.2.0     │ Apache Karaf :: Decanter :: Marshaller :: Raw
 32 │ Active │  80 │ 2.2.0     │ Apache Karaf :: Decanter :: Parser :: Identity
 33 │ Active │  80 │ 2.2.0     │ Apache Karaf :: Decanter :: Parser :: Regex
 34 │ Active │  80 │ 2.2.0     │ Apache Karaf :: Decanter :: Parser :: Split
 38 │ Active │  80 │ 4.3.0.RC1 │ Apache Karaf :: OSGi Services :: Event
 57 │ Active │  80 │ 4.14.0    │ Apache XBean OSGI Bundle Utilities
 58 │ Active │  80 │ 4.14.0    │ Apache XBean :: Classpath Resource Finder
 87 │ Active │  80 │ 1.0.4     │ JSR 353 (JSON Processing) Default Provider
 92 │ Active │  80 │ 7.2.0     │ org.objectweb.asm
 93 │ Active │  80 │ 7.2.0     │ org.objectweb.asm.commons
 94 │ Active │  80 │ 7.2.0     │ org.objectweb.asm.tree
108 │ Active │  80 │ 0.0.0     │ profiles

karaf@root()> http:list
ID │ Servlet       │ Servlet-Name   │ State       │ Alias             │ Url
───┼───────────────┼────────────────┼─────────────┼───────────────────┼──────────────────────
28 │ RestCollector │ ServletModel-2 │ Deployed    │ /decanter/collect │ [/decanter/collect/*]

karaf@root()>                                                                               
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
            <image>adoptopenjdk/openjdk11:alpine-jre</image>
        </from>
        <to>
            <image>fpaponapache.azurecr.io/fpaponapache/karaf-custom-distrib:${project.version}</image>
            <auth>
                <username>fpaponapache</username>
                <password></password>
            </auth>
        </to>
        <container>
            <entrypoint>
                <entrypoint>/karaf/bin/karaf</entrypoint>
                <entrypoint>run</entrypoint>
            </entrypoint>
            <workingDirectory>/karaf</workingDirectory>
            <ports>
                <port>8101</port>
                <port>1099</port>
                <port>44444</port>
                <port>8181</port>
            </ports>
        </container>
        <extraDirectories>
            <paths>
                <path>target/assembly</path>
            </paths>
            <permissions>
                <permission>
                    <file>/karaf/bin/karaf</file>
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

We define the Karaf instance entrypoint to the  `karaf run` and expose the ports:

*  8101 for ssh
*  1099 and 44444 for JMX
*  8181 for http

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
fpaponapache.azurecr.io/fpaponapache/karaf-custom-distrib   1.0.0-SNAPSHOT        8ed6cc4bae93        2 days ago          230MB
```

Now you can run a container:

```
docker run -it --name karaf-custom fpaponapache.azurecr.io/fpaponapache/karaf-custom-distrib:1.0.0-SNAPSHOT
```

Find the IP of the container:

```bash
docker inspect karaf-custom

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

Check the availability of the Karaf Decanter rest collector:

```bash
curl --header "Content-Type: application/json" --request POST --data '{"message":"it works","level":"info"}' http://172.17.0.2:8181/decanter/collect

* Expire in 0 ms for 6 (transfer 0x558fd1d68f50)
*   Trying 172.17.0.2...
* TCP_NODELAY set
* Expire in 200 ms for 4 (transfer 0x558fd1d68f50)
* Connected to 172.17.0.2 (172.17.0.2) port 8181 (#0)
> POST /decanter/collect HTTP/1.1
> Host: 172.17.0.2:8181
> User-Agent: curl/7.64.0
> Accept: */*
> Content-Type: application/json
> Content-Length: 37
> 
* upload completely sent off: 37 out of 37 bytes
< HTTP/1.1 201 Created
< Content-Length: 0
< Server: Jetty(9.4.22.v20191022)
< 
* Connection #0 to host 172.17.0.2 left intact
```

### Connecting with VisualVM on remote docker container

- Add a new remote host.
- Add a new JMX connection.

*Configuration*:
- Connection: `service:jmx:rmi://172.17.0.2:1099/jndi/rmi://172.17.0.2:1099/karaf-root`
- Display-name: `karaf-docker`
- Use security credentials: `username=karaf` / `password=karaf`

NB: at the end of the url connection, the karaf instance name is **root** and you can see your instance name in the `$KARAF_HOME/instances/instance.properties`

```
/karaf/instances # cat instance.properties 
count = 1
item.0.name = root
item.0.loc = /karaf
item.0.pid = 61
item.0.root = true
```

## Conclusion

You now have the basic information to create a Karaf custom distribution and pull it to a Docker registry.

The source of the example are available on Github [here](https://github.com/fpapon/blog-tutorial/tree/master/karaf-custom-distribution)
