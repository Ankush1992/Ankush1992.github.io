---
layout: post
title: Custom error pages with Spring Boot
categories: articles
tags: [spring-boot]
comments: true
---

If you are like me ,and loathe XML configurations with a passion , and have been using Spring MVC for a while or deciding which framework to use for your next project , then you are in for a treat. Spring Boot is a great web framework based on the philosophy of convention over configuration , backed by the goodness of the entire Spring ecosystem ,and does not mandate a single line of XML. How about that ! 

One of the regular requirement for any website / web app is to built custom error pages. These pages can be plain white pages that only show the error message with its Http error code (which is what are going to do ) or some fancy page displaying the error message.

Here is the directory structure :

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

Spring Boot starters (spring-boot-starter-web in particular) comes with an embedded tomcat container. When any exception is thrown by our handler methods that we DO NOT CATCH IMPLICITLY ,  the web container would look for ways to handle this exception . It looks for such configurations  inside the web.xml file .

We are routing tomcat to the  `/errors` mapping in such an event. As error codes are something very generic to an application,it is a good idea to create a whole new Controller for it.

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
{% endhighlight %}

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

Run the program . Type any random URL that comes to mind(that is not defined as a RequestMapping) .You should see the following:

![customerrorpage404](https://cloud.githubusercontent.com/assets/7692552/10869183/fcb60720-80cc-11e5-87d9-0b5eda78e4ec.png "404")

And when you go over to `/home` , the handler method would throw an Exception,which is then handled  by the Dispatcher Servlet,which routes the error to `/error`. 

![customerrorpage500](https://cloud.githubusercontent.com/assets/7692552/10869195/4d1b6a2a-80cd-11e5-94c6-94ebd9bf968a.png "500")




Download the sample  [here](https://github.com/Ankush1992/sample-boot-error-pages).

