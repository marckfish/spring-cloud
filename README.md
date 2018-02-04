If you want known how config-server does work, go to the README file in the config-server module.

** Quickstart **

Don't forget to clone *https://github.com/marckfish/config-resources*

- mvn clean install
- java -jar config-server/target/config-server.jar
- java -jar config-server/target/registry-service.jar
- java -jar config-server/target/edge-service.jar
- java -jar config-server/target/messages-service.jar

Go to eureka home page to see the registred services *http://localhost:8761/*

To show the welcome message go to http://localhost:8765/messages/hello
