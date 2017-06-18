# spring-cloud-config
As a reminder, spring cloud config server is an externalized configuration server, it allows to centralize and manage all properties for applications in distributed systems.  

This project shows how to use the config server in real conditions.  

I took a simple example, there are two modules in this project, one is the config-server,the second is a service reading configuration files from the config server. It demonstrates how to change propertie file without rebuild or restart the service.

**Environment**

- Java 8
- Spring boot 1.5.4
- Maven 3.9.9

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

The Config-server will run on port 8888 (This is the default port, we can remove server.port, 
the server will still run on 8888)

spring.cloud.config.server.git.uri allows you to specify the path to the Config Server's Git Repository which contains all the properties files, 
in our case we pointed to a local repository.

To point to remote repository replace _file://${user.home}/github/config-resources_ by https://github.com/marckfish/config-resources

Note: If you use a local repository, don't forget to clone https://github.com/marckfish/config-resources_, and put it for example in /user/home/github directory.

As we didn't implement the security, we have disabled it.

The repository https://github.com/marckfish/config-resources contains for the moment only one file: "messages-dev.yml"

**Running the config-server**

Go to config-server directory and run the following commands:

1- mvn clean install <br/>
2- java -jar target/config-server <br/>
3- Go to http://localhost:8888/env to see if the server is running. 
If it is, you will see all the environment configuration. As a reminder /env is an actuator endpoint. <br/>
4- To see the content of messages-dev.yml go to http://localhost:8888/messages/dev/master, the browser will display for you the following content   
    
    {"name":"messages","profiles":["dev"],"label":"master","version":"72f0913970caded8443f49ab0bdadf13b1f9e304","state":null,"propertySources":[{"name":"file:///home/merzouk/github/config-resources/messages-dev.yml",
    "source":{"message.hello":"Hello everybody!","spring.rabbitmq.host":"localhost","spring.rabbitmq.username":"guest","spring.rabbitmq.password":"guest","spring.rabbitmq.port":5672,"server.port":"${port:8569}"}}]}
    
**Understand *http://localhost:8888/messages/dev/master* URL**

1- **_messages_** is the logical name of the application, it must be unique. To set a logical name, use spring.application.name in bootstrap.yml located in the messages module, 
we will explain this file in the next section. In fact The config server will use the name to resolve and pick up appropriate properties from the config server repository.<br/>
2- **_dev_** is the profile, we can have a several profiles, the properties file name pattern is: {logical-name}-{profile}.yml(properties),in our case we have got just one _dev_ profile, the file is messages-dev. 
    The default profile's name is: _default_. if we had just messages.yml without specific profile, the url would have been *http://localhost:8888/messages/default/master*<br/>
3- **_master_** is the label, master is the default value for this label.  This is an optional Git Label.<br/>

Finally, the pattern of the url is like this : _http://localhost:8888/{logical-name}/{profile}/{optional-label}_

**II- Access to the config server from clients (messages module)**

The messages service which uses the Config server is called a Config client.

1- So, to use messages as a config client,we've added the following dependencies:

    <dependency>
    			<groupId>org.springframework.boot</groupId>
    			<artifactId>spring-boot-starter-actuator</artifactId>
    		</dependency>
    		<dependency>
    			<groupId>org.springframework.cloud</groupId>
    			<artifactId>spring-cloud-config-server</artifactId>
    		</dependency>

- Actuator is mandatory for to allow the refresh of the configuration properties.

2- We've added the following dependency to include the Spring Cloud dependencies:

        <dependencyManagement>
    		<dependencies>
    			<dependency>
    				<groupId>org.springframework.cloud</groupId>
    				<artifactId>spring-cloud-dependencies</artifactId>
    				<version>Camden.SR6</version>
    				<type>pom</type>
    				<scope>import</scope>
    			</dependency>
    		</dependencies>
    	</dependencyManagement>

**Explaining bootstrap.yml**

    spring:
       application:
         name: messages
       profiles:
         active: dev
       cloud: #Config server
         config:
           uri: http://localhost:8888
    management:
      security:
        enabled: false
        
At startup, the service will send a query to the spring config server at http://localhost:8888, 
and will ask it to retrieve the right property file according to the name _messages_ and with the profile _dev_ (messages-dev) 

**Content of messages-dev.yml**

    message:
      hello: Hello everybody!
    
    spring:
      rabbitmq:
        host: localhost
        username: guest
        password: guest
        port: 5672
    
    server:
      port: ${port:8569}

The service is running on port 8569, we can override the port with the --port=xxxx option. 
Right now you can ignore the rabbitmq bloc, I will come back to it later.

To demonstrate the centralized configuration of properties and propagation of changes, we've added message.hello.

The service contains a Rest Controller "MessageController", we've added the "@RefreshScope" annotation, 
this annotation allows properties to be refreshed when there are changes.

        @RestController
    	@RefreshScope
    	@RequestMapping
    	public class MessageController{
    
    		@Value("${message.hello:hello}")
    		String message;
    
    		@GetMapping(path = "/hello")
    		public String getMessage(){
    			return message;
    		}
    	}
    	
**Quickstart**

Go to messages directory and run these commands:

Before all, check if the Config server is running, if it's not, please check the section "Running the config-server".

**Running the config-server**.

1- mvn clean install <br/>
2- java -jar target/messages-service.jar <br/>
3- Go to http://localhost:8569/hello, the browser will display "hello everybody!" <br/>

Now modify message.hello within message-dev.yml, replace "Hello everybody!" by "Hello world!"

If you refresh http://localhost:8888/messages/dev/master, the browser will display :

    {"name":"messages","profiles":["dev"],"label":"master","version":"72f0913970caded8443f49ab0bdadf13b1f9e304","state":null,"propertySources":[{"name":"file:///home/merzouk/github/config-resources/messages-dev.yml",
        "source":{"message.hello":"Hello world!","spring.rabbitmq.host":"localhost","spring.rabbitmq.username":"guest","spring.rabbitmq.password":"guest","spring.rabbitmq.port":5672,"server.port":"${port:8569}"}}]}
        
If you refresh http://localhost:8569/hello, the browser will always display "Hello everybody!", 
it's normal because the configuration properties was not reloaded yet. 
To reload the configuration properties, call the /refresh of the messages service.

Execute the following command:

    curl -d {} localhost:8569/refresh
    
The bellow command send an empty POST to /refresh. Now if you refresh _http://localhost:8569/hello_, you will see "Hello world!".

**Spring Cloud Bus**

The bellow approach allows to refresh configuration parameters without restarting the service. This is useful if we have 1 or 2 instances of the service running. <br/>
If we have more than 5 instances for example, we have to call /refresh for each instance. it's cumbersome to do this.

For that we will use Spring Cloud Bus, which provides a mechanism to refresh configurations across multiple instances without knowing how many instances there are, or their locations.<br/>
The mechanism is simple to understand. In fact each instance connect to a single message broker. Each instance subscribes for change events, and it refreshes local configuration when necessary. 

The refresh is triggered by a call to "/bus/refresh" endpoint by any instance, then the changes are propagated to other instances through the cloud bus and the single common message broker.

In our case RabbitMQ is used as the AMQP message broker. To implement it, we've added the following dependency:

            <dependency>
    			<groupId>org.springframework.cloud</groupId>
    			<artifactId>spring-cloud-starter-bus-amqp</artifactId>
    		</dependency>

before to continue, install rabbitmq server **https://www.rabbitmq.com/download.html** <br/>

To verify if the server is running, go to http://localhost:15672, username: guest, password: guest

Reset the message.hello property, then rebuild and restart the messages service:

- java -jar target/messages-service.jar --port=8569
- java -jar target/messages-service.jar --port=8570

Now two instances are running one on port 8569, and another one on port 8570.

Return to http://localhost:8569/hello and http://localhost:8570/hello, and now the browser will display "Hello world!".

Change message.hello to "Hello world!" and call /bus/refresh:
    
    curl -d {} http://localhost:8569/bus/refresh?destination=messages:**
                        
                                    or
                                    
    curl -d {} http://localhost:8569/bus/refresh
    
For both instance you will see:

    o.s.cloud.bus.event.RefreshListener : Received remote refresh request. Keys refreshed [message.hello]

Return to http://localhost:8569/hello and http://localhost:8570/hello, and now the browser display "Hello world!".

**Source**

- https://spring.io/guides/gs/centralized-configuration/
- https://github.com/spring-cloud/spring-cloud-config