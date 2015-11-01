---
layout: post
title: Custom error pages with Spring Boot
categories: articles
tags: [spring-boot]
comments: true
---

If you are like me ,and loathe XML configurations with a passion , and have been using Spring MVC for a while or deciding which framework to use for your next project , then you are in for a treat. Spring Boot is a great web framework based on the philosophy of convention over configuration , backed by the goodness of the entire Spring ecosystem ,and does not mandate a single line of XML. How about that ! 

One of the regular requirement for any website / web app is to built custom error pages. These pages can be plain white pages that only show the error message with its Http error code (which is what are going to do ) or some fancy page displaying the error message.


![directory](https://cloud.githubusercontent.com/assets/7692552/10869162/ec15e896-80cb-11e5-8eaa-27161194aeb8.png "Directory")


Consider the following snippet :

{% highlight java %}

@SpringBootApplication
public class SampleBootErrorPagesApplication {
	
   private static final String LOCATION = "/errors";

   public static void main(String[] args) {
       SpringApplication.run(SampleBootErrorPagesApplication.class, args);
   }
    
    @Bean
    public EmbeddedServletContainerCustomizer containerCustomizer() {
      return (container -> {
   	   //route all errors towards /error .
   	   final ErrorPage errorPage=new ErrorPage(LOCATION);
   	   container.addErrorPages(errorPage);
      });
   }
 }
{% endhighlight %}
