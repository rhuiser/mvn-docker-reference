FROM 955414570717.dkr.ecr.eu-west-1.amazonaws.com/myorg/myproject-base-java:8u111-b14

ADD spring-cloud-config-server-1.0.4.RELEASE.jar /
ADD configserver.yml /

CMD ["java", "-Xms64M", "-Xmx256M", "-jar", "spring-cloud-config-server-1.0.4.RELEASE.jar"]

EXPOSE 8888
