---
layout: post
title: Circuit Breaker with Hystrix and Spring Boot(Part I)
categories: spring-boot,hytrix,microservices,circuit-breaker
tags: [spring-boot,distributed-computing]
comments: true
---

Evolving from a single-node setup/Monolith architecture to a multi-node / Microservices or distributed set up is always tricky.
All is honky dory and dandy in a single node architecture.However,one of the first disasters you will ever face as you shift to a distributed setup is Cascading failure when one of your services shuts down or responds very slowly to requests . In turn , the fleet of services depending on this service will stop responding . 
We circumvent this problem by using the Circuit Breaker Pattern.Martin Fowler has a fantastic [article](http://martinfowler.com/bliki/CircuitBreaker.html) explaining the concept.

# Scenario

We will run two apps on localhost , one running on port 8080 and the other on 8081 . The one running on port 8080 will be called `Node A` and the one running on port 8081 `Node B` . For every requests that Node A receives ,it makes a `GET` requests to Node B .

#### Case I : Both Node A and Node B are running.

Both of our nodes are running normally . 
We have 2 Controllers, one for our caller ,Node A , and one for our remote service,Node B.


```java
@RestController
public class NodeAController {
	
	@GetMapping("NodeA")
	public String makeCallToNodeB() throws Exception{
		return new RestTemplate()
				.getForEntity(new URI("http://localhost:8081/NodeB"), String.class)
				.getBody();
	}
}
```

```java
@RestController
public class NodeBController {

	@GetMapping("NodeB")
	public String handleNodeARequest(){
		return "Successfully returning value from Node B";
	}
}
```

Open your browser and go over to `localhost:8080/NodeA` . Since both services are running fine , Node B responds as per usual.

![localhost](https://cloud.githubusercontent.com/assets/7692552/19620808/19719260-98a2-11e6-9fb2-a9accd3154ca.png "localhost")

#### Case II: Node A is running , but Node B is down / timing out.

Keep running Node A and shut off Node B. 
![localhost](https://cloud.githubusercontent.com/assets/7692552/19620836/9430a8ec-98a2-11e6-91d2-26f38f71129e.png "localhost")

That's no good ! We want our Node A to keep running and ideally return something in case Node B is down.
This is where Hystrix comes in . [Here])(https://github.com/Netflix/Hystrix/wiki) is an excellent documentation explaining Hystrix in depth.

We set up Hystrix to monitor the calls from Node A to Node B , and execute a fallback method whenever Node B starts to lag.

Firstly , you need to tell Spring to enable Hystrix : 

```java
@SpringBootApplication
@EnableCircuitBreaker
public class NodeAMainApplication {

	public static void main(String[] args) {
		SpringApplication.run(NodeAMainApplication.class, args);
	}
}
```

Next, you need to annotate Node A's method that makes the web service call with `@HystrixCommand` .

```java
@RestController
public class NodeAController {
	
	@GetMapping("NodeA")
	@HystrixCommand(fallbackMethod="fallback")
	public String makeCallToNodeB() throws Exception{
		return new RestTemplate()
				.getForEntity(new URI("http://localhost:8081/NodeB"), String.class)
				.getBody();
	}
	
	public String fallback(){
		return "Red alert! This String is returned by the fallback method";
	}
}
```

Now go on over to `localhost:8080/NodeA` . You will see :
![node a -node b-off](https://cloud.githubusercontent.com/assets/7692552/19620894/01fc5956-98a4-11e6-907c-06a833bef879.png)

Very good. Lets consider another more practical scenario. We want to trip the circuit when Node B times out more than 2 times on a 10 second sliding window. We set the read time out to 2 seconds .

```java
@RestController
public class NodeAController {
	
	@GetMapping("NodeA")
	@HystrixCommand(
			fallbackMethod="fallback",
			commandProperties = {
					@HystrixProperty(name="execution.isolation.thread.timeoutInMilliseconds",value="1600"),
					@HystrixProperty(name="circuitBreaker.requestVolumeThreshold",value="2"),
					@HystrixProperty(name="circuitBreaker.sleepWindowInMilliseconds",value="10000")
				}
			)
	public String makeCallToNodeB() throws Exception{
		return new RestTemplate()
				.getForEntity(new URI("http://localhost:8081/NodeB"), String.class)
				.getBody();
	}
	
	public String fallback(){
		return "Red alert! This String is returned by the fallback method";
	}
}
```

That's it folks. View the complete set of configurations [here](https://github.com/Netflix/Hystrix/wiki/Configuration) .



