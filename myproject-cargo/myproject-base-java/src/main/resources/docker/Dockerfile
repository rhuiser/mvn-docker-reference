FROM centos:7
MAINTAINER Robin Huiser "robin.huiser@rdc.pt"

# Version of JDK 8
ENV JDK_VERSION ${jdk-image.version}
ENV JDK_BUILD_VERSION ${jdk-image.build}

# Set correct environment variables.
ENV	HOME /root
ENV	LANG en_US.UTF-8
ENV	LC_ALL en_US.UTF-8
ENV JAVA_HOME /usr/java/default

# We are not running update all for yum -- we want a repeatable outcome
RUN yum install -y curl unzip

# Install Java
RUN cd /tmp && \
    curl -sLO "http://download.oracle.com/otn-pub/java/jdk/$JDK_VERSION-$JDK_BUILD_VERSION/jdk-$JDK_VERSION-linux-x64.rpm" \
    -H 'Cookie: oraclelicense=accept-securebackup-cookie' && \
      rpm -i jdk-$JDK_VERSION-linux-x64.rpm && \
      rm -f jdk-$JDK_VERSION-linux-x64.rpm && \
      yum clean all

# Install strong encryption pack
RUN cd /tmp && \
    curl -sLO "http://download.oracle.com/otn-pub/java/jce/8/jce_policy-8.zip" \
    -H "Cookie: oraclelicense=accept-securebackup-cookie" && \
      unzip -oj -d $JAVA_HOME/jre/lib/security jce_policy-8.zip \*/\*.jar && \
      rm jce_policy-8.zip

RUN mkdir -p /home/jrunner && \
    groupadd -r jrunner -g 433 && \
    useradd -u 431 -r -g jrunner -d /home/jrunner -s /sbin/nologin -c "Java Docker image user" jrunner && \
    chown -R jrunner:jrunner /home/jrunner

USER jrunner
