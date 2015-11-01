---
layout: post
title: Custom error pages with Spring Boot
categories: articles
tags: [spring-boot]
comments: true
---
If you are like me ,and loathe XML configurations with a passion , and have been using Spring MVC for a while or deciding which framework to use for your next project , then you are in for a treat. Spring Boot is a great web framework based on the philosophy of convention over configuration , backed by the goodness of the entire Spring ecosystem ,and does not mandate a single line of XML. How about that ! 

One of the regular requirement for any website / web app is to built custom error pages. These pages can be plain white pages that only show the error message with its Http error code (which is what are going to do ) or some fancy page displaying the error message.

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

Spring Boot starters (spring-boot-starter-web in particular) comes with an embedded tomcat container. When any exception is thrown by our handler methods that we DO NOT CATCH IMPLICITLY (a topic for another post) ,  the web container would look for ways to handle this exception . It looks for such configurations  inside the web.xml file .

We are routing tomcat to visit the `/errors` mapping in such an event. Let's go ahead and create the ErrorController.

{% highlight java %}
@Controller
public class ErrorController {

   @Autowired
   private ErrorService errorService;
	     
   @RequestMapping(value="/errors",method=RequestMethod.GET)
   public String renderErrorPage(final Model model,final HttpServletRequest request){
	   
	   //Get the Http error code.
	   final int error_code=getHttpStatusCode(request);
	   
	   //Generates Error message for corresponding Http Error Code.
	   final String error_message=errorService.generateErrorMessage(error_code);
	   
	   model.addAttribute("errorMsg",error_message);
	   
	   return "error";
   }  
	   
   private int getHttpStatusCode(final HttpServletRequest request){
	   return (int) request.getAttribute("javax.servlet.error.status_code");
   }
}
{% endhiglight %}

We capture the Http error code in the `error_code` variable , and create another service  to generate some error message based on the error code.

{% highlight java %}
@Service
@PropertySource("classpath:httpErrorCodes.properties")
public class ErrorService {
	
	@Autowired
	private Environment env;

	public String generateErrorMessage(final int error_code){
		 String message="";
		   switch(error_code){  
			   case 400: message=env.getProperty("400");
			   			 break;
			   case 401: message=env.getProperty("401");
			   			 break;
			   case 404: message=env.getProperty("404");
			   			 break;
			   case 500: message=env.getProperty("500");
			   			 break;
			   //etc 
			   //Put in all Http error codes here.
		   }
		   return message;
	}
}
{% endhighlight %}

I have  demonstrated with  only four error codes, but you get the idea  . We put error codes and error messages as key value pairs inside our property file. Place the property file inside the resources folder.

```
400=Bad Request.
401=Unauthorized
404=Not Found.
405=Method Not Allowed


500=Internal Server Error

```




Download the sample  [here](https://github.com/Ankush1992/sample-boot-error-pages).

here is a <q> q tag </q>
