# spring-cloud-config

This project shows how to use The config server in real conditions.

I take a simple example, there is two modules in this project, one's for config-server,the second for reading configuration files from config server, and demonstrate how to changes file properties without rebuild or restart the service

**I- Config Server**

This module is our server-config, his pom contains the following dependencies

            <dependency>
    			<groupId>org.springframework.boot</groupId>
    			<artifactId>spring-boot-starter-actuator</artifactId>
    		</dependency>
    		<dependency>
    			<groupId>org.springframework.cloud</groupId>
    			<artifactId>spring-cloud-config-server</artifactId>
    		</dependency>
    		<dependency>
    			<groupId>org.springframework.boot</groupId>
    			<artifactId>spring-boot-starter-web</artifactId>
    		</dependency>
    		<dependency>
    			<groupId>org.springframework.boot</groupId>
    			<artifactId>spring-boot-starter-test</artifactId>
    			<scope>test</scope>
    		</dependency>
   
    
**Understand bootstrap.yml**

In bootstrap.yml in src/main/resources, we have got the following configuration:

    server:
      port: 8888
    spring:
      cloud:
        config:
          server:
            git:
              uri: file://${user.home}/github/config-resources
      application:
        name: CONFIG-SERVER
    security:
      basic:
        enabled: false
    management:
      security:
        enabled: false

Config-server will run on port 8888 (This his port by default, we can remove server.port, 
the server will run always on 8888)

spring.cloud.config.server.git.uri Allows to specify the path to Config Server Git Repository which contains all file properties, in our case we pointed to local repository.
To point to remote repository replace _file://${user.home}/github/config-resources_ by https://github.com/marckfish/config-resources

As we not use yet the security, we disable it

The repository *https://github.com/marckfish/config-resources* contains for the moment one only file messages-dev.yml

**Running the config-server**
Go to config-server directory and take the following commands:

1- mvn clean install <br/>
2- java -jar target/config-server <br/>
3- Go to http://localhost:8888/env to see if the server is running. If is the case,
you will see all environment configurations. As a reminder /env is an actuator endpoint. <br/>
4- To see the content of messages-dev.yml go to http://localhost:8888/messages/dev/master, the browser will display for you the following content   
    
    {"name":"messages","profiles":["dev"],"label":"master","version":"72f0913970caded8443f49ab0bdadf13b1f9e304","state":null,"propertySources":[{"name":"file:///home/merzouk/github/config-resources/messages-dev.yml",
    "source":{"message.hello":"Hello everybody!","spring.rabbitmq.host":"localhost","spring.rabbitmq.username":"guest","spring.rabbitmq.password":"guest","spring.rabbitmq.port":5672,"spring.cloud.discovery.enabled":true,"spring.cloud.discovery.serviceId":"CONFIG-SERVER","server.port":"${port:8569}"}}]}
    
**Understand *http://localhost:8888/messages/dev/master* URL**

1- **_messages_** is the logical name of the application, it must be unique. To set a logical name, use spring.application.name in bootstrap.yml located in the messages module, 
we will explain this file in the next section. In fact The config server will use the name to resolve and pick up appropriate properties from the config server repository.<br/>
2- **_dev_** is the profile, we can have a severals profiles, the file properties id named like {logical-name}-{profile}.yml(properties),in our case we have got just one profile _dev_, the file is named messages-dev. 
    The default profile is named _default_. if we had just messages.yml without specific profile, the url will have been *http://localhost:8888/messages/default/master*<br/>
3- **_master_** is the label, is named master by default. The label si an optional Git Label.<br/>

Finally, the pattern of the url is like this : _http://localhost:8888/{logical-name}/{profile}/{optional-label}_

**II- Access to the config server from clients (messages module)  to be continued**
