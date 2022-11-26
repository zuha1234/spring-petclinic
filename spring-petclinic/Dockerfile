FROM tomcat:7
COPY target/petclinic.war /usr/local/tomcat/webapps/
COPY server.xml /usr/local/tomcat/conf/
EXPOSE 8080
RUN chmod +x /usr/local/tomcat/bin/catalina.sh
CMD ["catalina.sh", "run"]
