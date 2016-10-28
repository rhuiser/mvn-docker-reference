FROM ${docker.repo}/${docker.java-base}

ADD ${service.artefactId}-${service.version}.jar /

CMD ["java", "-Xms64M", "-Xmx256M", "-jar", "${service.artefactId}-${service.version}.jar"]

EXPOSE 8080