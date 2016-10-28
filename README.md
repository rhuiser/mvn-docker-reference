# Docker Cargo project
Dockers ship stuff - that we all know. It makes sense (if we keep the analogy with real ships and cargo), the cargo itself is **not** manufactured in the harbour, but retrieved from elsewhere by boat or train (read: artifactory such as Nexus).

That is why there is a split between compiling, testing and code distribution (to artifactory) and packaging your delivered artefacts into containers. Hence... cargo.

Here I will provide you with an overview how to create such a cargo project and have it orchestrated by maven so it can be included into any standard build server platform as an additional step.

This document will show you how to: 

* setup your development Docker host
* create the Maven project, incl. structure of directories
* build, test and deploy your Docker image
* keep your Docker development host tidy and some Docker debugging options 


## Prerequisits
* Docker 1.12 or higher installed for your platform
* VirtualBox 5.1.2 or higher
* Maven 3.3.x
* AWS Commandline Tools and configured key/secret if you plan to deploy to ECR


## Create your development Docker host
Set-up your Docker environment on your workstation; either create or resume an existing instance here.

~~~sh
# Create the Docker host in VirtualBox
docker-machine create --driver virtualbox igtb-digital-dockerhost

# ... or resume
docker-machine start igtb-digital-dockerhost

# Set your Docker environment variables
eval $(docker-machine env igtb-digital-dockerhost)
~~~

# Set-up overall Docker cargo modules
The basis of a cargo module is quite simple:

~~~
digital-cargo/
├── ReadMe.md
├── digital-helloworld-service-docker
│   ├── pom.xml
│   └── src
│       └── main
│           └── resources
│               └── docker
│                   └── Dockerfile
├── digital-parent
│   └── pom.xml
└── pom.xml
~~~                

* 01-12: Cargo module (may include multiple cargo projects)
* 10-11: Parent project
* 03-09: A Cargo project
 

## Cargo project
The cargo project POM defines **build resources**, **project dependencies** and **plugin configuration** for Docker:

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">

    <parent>
        <artifactId>digital-parent</artifactId>
        <groupId>com.igtb.digital</groupId>
        <version>1.0.0-SNAPSHOT</version>
        <relativePath>../digital-parent/pom.xml</relativePath>
    </parent>
    <modelVersion>4.0.0</modelVersion>
   
    <artifactId>digital-helloworld-service-docker</artifactId>
    <name>iGTB :: HelloWorld Service</name>
  
    <properties>
		<!-- Docker image tag defaults to project version -->
		<image.version>${project.version}</image.version>
		
		<!-- Set the Spring boot service artefact properties here --> 
        <service.groupId>com.igtb.digital.backend</service.groupId>
        <service.artefactId>hello-world-service</service.artefactId>
        <service.version>0.1.0</service.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>${service.groupId}</groupId>
            <artifactId>${service.artefactId}</artifactId>
            <version>${service.version}</version>
            <type>jar</type>
        </dependency>
    </dependencies>

    <build>
        <resources>
            <resource>
                <directory>src/main/resources</directory>
                <filtering>true</filtering>
                <targetPath>${project.build.directory}</targetPath>
            </resource>
        </resources>
        <plugins>
            <plugin>
                <artifactId>maven-dependency-plugin</artifactId>
                <executions>
                    <execution>
                        <id>copy</id>
                        <phase>prepare-package</phase>
                        <goals>
                            <goal>copy</goal>
                        </goals>
                        <configuration>
                            <artifactItems>
                                <artifactItem>
                                    <groupId>${service.groupId}</groupId>
                                    <artifactId>${service.artefactId}</artifactId>
                                    <type>jar</type>
                                    <outputDirectory>${project.build.directory}/docker</outputDirectory>
                                    <destFileName>${service.artefactId}-${service.version}.jar</destFileName>
                                </artifactItem>
                            </artifactItems>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
            <plugin>
                <groupId>com.spotify</groupId>
                <artifactId>docker-maven-plugin</artifactId>
                <configuration>
                    <imageName>${docker.repo}/igtb/${project.artifactId}:${image.version}</imageName>
			        <imageTags>
			           <imageTag>${image.version}</imageTag>
			           <imageTag>latest</imageTag>
			        </imageTags>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
~~~


## Parent project
The parent POM defines **global variables** (versions for example) and **global dependencies** for all cargo projects:

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">

    <parent>
        <artifactId>digital</artifactId>
        <groupId>com.igtb.digital</groupId>
        <version>1.0.0-SNAPSHOT</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>
    <packaging>pom</packaging>

    <artifactId>digital-parent</artifactId>

    <properties>
		<!-- Docker repository here -->
		<docker.repo>955414570717.dkr.ecr.eu-west-1.amazonaws.com</docker.repo>
		
		<!-- Java Docker image and version to use -->
		<docker.java-base>igtb/digital-base-java:8u11-b12</docker.java-base>
		
		<!-- Commanon features and libs here -->
        <features-maven-plugin.version>2.3.4</features-maven-plugin.version>
        <springcloudconfigserver.version>1.0.0.RELEASE</springcloudconfigserver.version>
        <mysql-connector.version>5.1.23</mysql-connector.version>
    </properties>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>mysql</groupId>
                <artifactId>mysql-connector-java</artifactId>
                <version>${mysql-connector.version}</version>
            </dependency>
        </dependencies>
    </dependencyManagement>

</project>
~~~


## Cargo modules
The cargo POM defines **profiles**, **repositories** and **plugins** (amongst the Docker plugins from Spotify).

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    
    <groupId>com.igtb.digital</groupId>
    <artifactId>digital</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <packaging>pom</packaging>
	
	<modelVersion>4.0.0</modelVersion>
    <name>iGTB :: Digital Cargo</name>

    <prerequisites>
        <maven>3.2.1</maven>
    </prerequisites>

    <properties>
        <!-- unify the encoding for all the modules -->
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>

        <jdk.version>1.8</jdk.version>
        <compiler.fork>false</compiler.fork>
    </properties>

    <profiles>
        <profile>
            <id>default</id>
            <activation>
                <activeByDefault>true</activeByDefault>
            </activation>

            <modules>
                <module>digital-helloworld-service-docker</module>
            </modules>
        </profile>

        <profile>
            <id>backend</id>
            <activation>
                <activeByDefault>false</activeByDefault>
            </activation>

            <modules>
                <module>digital-helloworld-service-docker</module>
            </modules>
        </profile>
    </profiles>

    <repositories>
        <repository>
            <id>iGTB Artifact Repository</id>
            <url>https://artifacts.igtb.com/repo</url>
        </repository>
        <repository>
            <id>iGTB Artifact Snapshots Repository</id>
            <name>artifacts.igtb.com-snapshots</name>
            <url>https://artifacts.igtb.com/digital-development-snapshots</url>
            <snapshots>
                <enabled>true</enabled>
            </snapshots>
        </repository>
        <repository>
            <id>iGTB Artifact Releases Repository</id>
            <name>artifacts.igtb.com-releases</name>
            <url>https://artifacts.igtb.com/digital-development-internal-releases</url>
            <releases>
                <enabled>true</enabled>
            </releases>
            <snapshots>
                <enabled>false</enabled>
            </snapshots>
        </repository>
    </repositories>

    <build>
        <defaultGoal>install</defaultGoal>
        <pluginManagement>
            <plugins>
                <!-- This plugin is to be used for running dockers containers -->
                <plugin>
                    <groupId>org.jolokia</groupId>
                    <artifactId>docker-maven-plugin</artifactId>
                    <version>0.10.5</version>
                </plugin>
                <!-- This plugin is to be used for building docker images -->
                <plugin>
                    <groupId>com.spotify</groupId>
                    <artifactId>docker-maven-plugin</artifactId>
                    <version>0.4.13</version>
                    <executions>
                        <execution>
                            <id>build-image</id>
                            <phase>package</phase>
                            <goals>
                                <goal>build</goal>
                            </goals>
                        </execution>
                    </executions>
                    <configuration>
                        <dockerDirectory>${project.build.directory}/docker</dockerDirectory>
                    </configuration>
                </plugin>
                <plugin>
                    <artifactId>maven-antrun-plugin</artifactId>
                    <version>1.8</version>
                </plugin>
                <plugin>
                    <artifactId>maven-dependency-plugin</artifactId>
                    <version>2.10</version>
                </plugin>
            </plugins>
        </pluginManagement>
    </build>
</project>
~~~


# Build, test and deploy
This is all implemented using Maven (to keep your brain free of lots of complicated Docker commands). And yes... also to allow an automatic build server to do the magic.

## Build
This step will build the Docker image and publishes it to the local Docker host repository.

~~~sh
# Change into the directory of the cargo project and build
mvn clean package
~~~

## Test
This step will instantiate a container of the created Docker image.

~~~sh

~~~

## Deploy
Once you are happy, deploy the Docker container to the central repository so your fellow developers can use it. 

**Note** as a developer, you will be only pushing `latest` -- which means **SNAPSHOT**. This is only meant to provide  

### Using AWS Elastic Container Repository (ECR)
Login to ECR; update the `region` parameter as required.

~~~sh
# Evaluate (execute) the output of the aws command
eval $(aws ecr get-login --region eu-west-1)
~~~

**Note**: *make sure the repository exists in ECR before publishing, if not create one:*

~~~sh
# For example, for igtb/java-hello-world
aws ecr create-repository \
   --repository-name igtb/java-hello-world

# Optionally -- make it world-readable for pulling images by 3rd parties
aws ecr set-repository-policy \
   --repository-name igtb/java-hello-world \
   --policy-text '{
    "Version": "2008-10-17",
    "Statement": [
        {
            "Sid": "PublicAccess",
            "Effect": "Allow",
            "Principal": "*",
            "Action": [
                "ecr:GetDownloadUrlForLayer",
                "ecr:BatchGetImage",
                "ecr:BatchCheckLayerAvailability"
            ]
        }
    ]
}'
~~~

Next, publish your Docker:

~~~sh
mvn docker:build -Dimage.version=1.0.1-RELEASE \
   -DpushImageTag -DuseConfigFile=true
~~~

# Tips and tricks
Some handy Docker commands I'd like to use to keep my Docker host clean and to troubleshoot.

## Remove old processes
All instantiated containers leave some traces behind as can be seen once issued the command `docker ps -a`; to clean these finsihed processes up, issue:

~~~sh
docker ps -a | awk 'NR > 1 {print $1}' | xargs docker rm
~~~

## Remove old images
When building and testing, image versions are kept in track of in your local Docker host - hence consuming disk space as can be seen once issued the command `docker images -a`; to clean these intermediary images up, issue:

~~~sh
docker images | grep "<none>" | awk '{print $3}' | xargs docker rmi
~~~

## Attach to a running container
What is happening inside of my running Docker container!?!? Well... attach a console to the container and see for yourself :-)

~~~sh
docker exec -it [CONTAINER ID] bash
~~~

## Attach to a stopped container 
So... you have build your container, but it does not start. Or you want to inspect what all the steps have done with the local filesystem / config of your container. Command below will start your Docker with a shell (not running the final command):

~~~sh
docker run -it [IMAGE ID] bash
~~~

## View stdout of container
If configured well (and for sure you did), the container logs all to `stdout`... but it was started in the background (also well done!). Here is to get the logs from your running instance:

~~~sh
 docker logs [CONTAINER ID]
~~~

# Known Issues
Some issues are known when building with maven - most can be solved easily.

## Exception caught: system properties: docker has type STRING rather than OBJECT
Because the plugin uses Maven properties named like docker.build.defaultProfile, if you declare any other Maven property with the name docker you will get a rather strange-looking error from Maven:

	[ERROR] Failed to execute goal com.spotify:docker-maven-plugin:0.0.21:build (default) on project <....>: 
	Exception caught: system properties: docker has type STRING rather than OBJECT

**To fix this, rename the docker property in your pom.xml**.

## InternalServerErrorException: HTTP 500 Internal Server Error

Problem: when building the Docker image, Maven outputs an exception with a stacktrace like:

~~~
Caused by: com.spotify.docker.client.shaded.javax.ws.rs.InternalServerErrorException: HTTP 500 Internal Server Error
docker-maven-plugin communicates with your local Docker daemon using the HTTP Remote API and any unexpected errors that the daemon encounters will be reported as 500 Internal Server Error.
~~~

Check the Docker daemon log (typically at `/var/log/docker.log` or `/var/log/upstart/docker.log`) for more details.

## Invalid repository name ... only [a-z0-9-_.] are allowed

One common cause of `500 Internal Server Error` is attempting to build an image with a repository name containing uppercase characters, such as if the <imageName> in the plugin's configuration refers to ${project.version} when the Maven project version is ending in `SNAPSHOT`.

Consider putting the project version in an image tag (instead of repository name) with the `<dockerImageTags>` configuration option instead.
