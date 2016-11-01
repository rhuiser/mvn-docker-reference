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
docker-machine create --driver virtualbox myorg-myproject-dockerhost

# ... or resume
docker-machine start myorg-myproject-dockerhost

# Set your Docker environment variables
eval $(docker-machine env myorg-myproject-dockerhost)
~~~

# Set-up overall Docker cargo modules
The basis of a cargo module is quite simple:

~~~
myproject-cargo/
├── ReadMe.md
├── myproject-helloworld-service-docker
│   ├── pom.xml
│   └── src
│       └── main
│           └── resources
│               └── docker
│                   └── Dockerfile
├── myproject-parent
│   └── pom.xml
└── pom.xml
~~~                

* **01-12: Cargo module** - *defines profiles, repositories and plugins*
* **10-11: Parent project** - *defines global variables and global dependencies for all cargo projects*
* **03-09: A Cargo project** - *defines build resources, project dependencies and plugin configuration for Docker*


# Build, test and deploy
This is all implemented using Maven (to keep your brain free of lots of complicated Docker commands). And yes... also to allow an automatic build server to do the magic.

## Build
This step will build the Docker image and publishes it to the local Docker host repository.

The overall Maven project will:

1. Build the **backend** Spring Boot based hello-world-api and publish as artefact to local Maven repository
2. Build the **cargo** Docker overlay image for Java 1.8
3. Compose the **cargo** hello-world service Docker based upon:
	* Docker overlay image Java 1.8 (which we just created)
	* Hello-world artefact (retrieved from local repository)

~~~sh
mvn clean install
~~~

## Run
This step will run the created images as Docker containers on your Dockerhost.

First, we define a simple `docker-compose file` where we start the HelloWorld API and Config-Server Dockers. 

~~~yaml
helloworld-api:
    image: myorg/myproject-service-helloworld-api
    ports:
        - "8500:8080"
    links:
        - config-server

config-server:
    image: 955414570717.dkr.ecr.eu-west-1.amazonaws.com/myorg/myproject-config-server:1.0.0-SNAPSHOT
~~~

* The HelloWorld API Docker is linked to the Config-Server, as a result, Config Server will start first.
* A port-mapping has been assigned to HelloWorld API, as a result `port 8080` (listening on internal Docker network) will be exposed to the outside world via the Docker host external IP address using `port 8500`

~~~sh
# Bring up the Docker(s)
cd myproject-cargo
docker-compose up -d

# Verify if everything is OK
docker-compose ps

# Test the set-up
curl http://$(docker-machine ip myorg-myproject-dockerhost):8500/
~~~

Hello Word!

## Deploy
Once you are happy, deploy the Docker container to the central repository so your fellow developers can use it.

### Using AWS Elastic Container Repository (ECR)
Login to ECR; update the `region` parameter as required.

~~~sh
# Evaluate (execute) the output of the aws command
eval $(aws ecr get-login --region eu-west-1)
~~~

**Note**: *make sure the repository exists in ECR before publishing, if not create one:*

~~~sh
# For example, for myorg/java-hello-world
aws ecr create-repository \
   --repository-name myorg/java-hello-world

# Optionally -- make it world-readable for pulling images by 3rd parties
aws ecr set-repository-policy \
   --repository-name myorg/java-hello-world \
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
