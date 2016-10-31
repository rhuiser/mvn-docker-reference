FROM ${docker.repo}/${docker.java-base}

ADD ${service.artefactId}-${service.version}.jar /
ADD configserver.yml /

CMD ["java", "-Xms64M", "-Xmx256M", "-jar", "${service.artefactId}-${service.version}.jar"]

EXPOSE 8888
