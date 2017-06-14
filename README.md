# spring-cloud-config
As a reminder, spring cloud config server is an externalized configuration server, it allows to centralize and manage all properties for applications in distributed system.
    
This project shows how to use The config server in real conditions.

I take a simple example, there is two modules in this project, one's for config-server,the second for reading configuration files from config server, and demonstrate how to changes file properties without rebuild or restart the service

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

Config-server will run on port 8888 (This his port by default, we can remove server.port, 
the server will run always on 8888)

spring.cloud.config.server.git.uri Allows to specify the path to Config Server Git Repository which contains all file properties, in our case we pointed to local repository.
To point to remote repository replace _file://${user.home}/github/config-resources_ by https://github.com/marckfish/config-resources

Note: If you use a local repository, don't forgot to clone _clone https://github.com/marckfish/config-resources_ and put it for example in /user/home/github directory

As we not use yet the security, we disable it.

The repository *https://github.com/marckfish/config-resources* contains for the moment one only file messages-dev.yml

**Running the config-server**
Go to config-server directory and take the following commands:

1- mvn clean install <br/>
2- java -jar target/config-server <br/>
3- Go to http://localhost:8888/env to see if the server is running. If is the case,
you will see all environment configurations. As a reminder /env is an actuator endpoint. <br/>
4- To see the content of messages-dev.yml go to http://localhost:8888/messages/dev/master, the browser will display for you the following content   
    
    {"name":"messages","profiles":["dev"],"label":"master","version":"72f0913970caded8443f49ab0bdadf13b1f9e304","state":null,"propertySources":[{"name":"file:///home/merzouk/github/config-resources/messages-dev.yml",
    "source":{"message.hello":"Hello everybody!","spring.rabbitmq.host":"localhost","spring.rabbitmq.username":"guest","spring.rabbitmq.password":"guest","spring.rabbitmq.port":5672,"server.port":"${port:8569}"}}]}
    
**Understand *http://localhost:8888/messages/dev/master* URL**

1- **_messages_** is the logical name of the application, it must be unique. To set a logical name, use spring.application.name in bootstrap.yml located in the messages module, 
we will explain this file in the next section. In fact The config server will use the name to resolve and pick up appropriate properties from the config server repository.<br/>
2- **_dev_** is the profile, we can have a severals profiles, the file properties is named like {logical-name}-{profile}.yml(properties),in our case we have got just one profile _dev_, the file is named messages-dev. 
    The default profile is named _default_. if we had just messages.yml without specific profile, the url will have been *http://localhost:8888/messages/default/master*<br/>
3- **_master_** is the label, is named master by default. The label si an optional Git Label.<br/>

Finally, the pattern of the url is like this : _http://localhost:8888/{logical-name}/{profile}/{optional-label}_

**II- Access to the config server from clients (messages module)**

The messages service use the Config server, it's a Config client.

1- So that, to use messages as a config client,we added the following dependencies:

    <dependency>
    			<groupId>org.springframework.boot</groupId>
    			<artifactId>spring-boot-starter-actuator</artifactId>
    		</dependency>
    		<dependency>
    			<groupId>org.springframework.cloud</groupId>
    			<artifactId>spring-cloud-config-server</artifactId>
    		</dependency>

- Actuator is obligatory for the refreshing the configuration properties.

2- We added the following dependency to include the Spring Cloud dependencies:

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
        
At the startup, the service will interrogate the spring config server at http://localhost:8888, 
and will tell it to retrieve the adequat property file with the name _messages_ and with the profile _dev_ (messages-dev)

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

The service turn on port 8569, we can override the port by --port=xxxx.
For the moment you can ignore the bloc rabbitmq, I will come back to that later.

To demonstrate the centralized configuration of properties and propagation of changes, we added message.hello.

the service contains a Rest Controller MessageController, we added the @RefreshScope annotation, 
this annotation allows properties to be refreshed when there in change.

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

Before all, check if the Config server is running, if this is not the case refer you to the section **Running the config-server**.

1- mvn clean install <br/>
2- java -jar target/messages-service.jar <br/>
3- Go to http://localhost:8569/hello, the browser will display "hello everybody!" <br/>

Now modify message.hello within message-dev.yml, replace "Hello everybody!" by "Hello world!"
If you refresh http://localhost:8888/messages/dev/master, the browser will display :

    {"name":"messages","profiles":["dev"],"label":"master","version":"72f0913970caded8443f49ab0bdadf13b1f9e304","state":null,"propertySources":[{"name":"file:///home/merzouk/github/config-resources/messages-dev.yml",
        "source":{"message.hello":"Hello world!","spring.rabbitmq.host":"localhost","spring.rabbitmq.username":"guest","spring.rabbitmq.password":"guest","spring.rabbitmq.port":5672,"server.port":"${port:8569}"}}]}
        
If you refresh http://localhost:8569/hello, the browser will display always "Hello everybody!", 
it's normal because the configuration properties was not reloaded yet.<br/>
To reload the configuration properties, call the _/refresh_ of the messages service. <br/>

Execute the following command:

    curl -d {} localhost:8569/refresh
    
The bellow command send an empty POST to /refresh. Now if you refresh _http://localhost:8569/hello_, you will see "Hello world!".

**Spring Cloud Bus**

The bellow approach allows to refresh configuration parameters without restarting the service. This is useful if we have 1 or 2 instances of the service running. <br/>
If we have more than 5 instances for example, we have to call /refresh for each instance. it's cumbersome to do this.

For that we will use Spring Cloud Bus, which provides a mechanism to refresh configurations across multiple instances without knowing how many instances there are, or their locations.<br/>
The mechanism is simple to understand. In fact each instance connect to a single message broker. Each instance subscribes for change events, and refreshes its local configuration when necessary. 

The refresh is triggered by a call "/bus/refresh" endpoint by an any instance, then the change is propagated to other instances through the cloud bus and the single common message broker. 

In our case RabbitMQ is used as the AMQP message broker. To implement it we added the following dependency:

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

Go to http://localhost:8569/hello and http://localhost:8570/hello, the browser will display "Hello everybody!" for both instances

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