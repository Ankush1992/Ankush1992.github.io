---
layout: post
title: Custom handler method resolvers in Spring boot.
categories: spring-boot
tags: [spring-boot,advanced]
comments: true
---

Ever wonder what happens under the hood whenever you use something like `@RequestParam` in your handler method inside your controllers ? We'll create an annotation `@UserAgent` , use this annotation inside the handler method and resolve it to a String.
Our end result will look like :

```java
@RestController
public class TestController {
	
	@GetMapping("test")
	public String resolveUserAgent(@UserAgent String userAgent){
		return "userAgent is : " + userAgent;
	}
	
}
```


We proceeed with the following plan:
* Create an annotation `@UserAgent` .
* Implement Spring's [HandlerMethodArgumentResolver](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/method/support/HandlerMethodArgumentResolver.html) . We call this class `UserAgentAnnotationMethodResolver` .
* Register  `UserAgentAnnotationMethodResolver` with [WebMvcConfigurerAdapter](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/servlet/config/annotation/WebMvcConfigurerAdapter.html)


Let's get to it.

```java
public class UserAgentAnnotationMethodResolver implements HandlerMethodArgumentResolver{

	@Override
	public Object resolveArgument(
			final MethodParameter methodParam,
			final ModelAndViewContainer modelAndViewContainer, 
			final NativeWebRequest webRequest,
			final WebDataBinderFactory webDataBinderFactory) throws Exception 
	{
		String result = "";
		if(webRequest.getNativeRequest() instanceof HttpServletRequest){
			final HttpServletRequest httpServletRequest = webRequest.getNativeRequest(HttpServletRequest.class);
			result = httpServletRequest.getHeader("User-Agent");
		}
		return result;
	}

	@Override
	public boolean supportsParameter(final MethodParameter methodParam) {
		return methodParam.hasParameterAnnotation(UserAgent.class);
	}
}

```

When Spring encounter's the annotation `@UserAgent` , it will scan its registry for an implementation of `HandlerMethodArgumentResolver` . But how the hell would it know which `HandlerMethodArgumentResolver` to use for our `@UserAgent` ? 
Notice the method `supportsParameter(final MethodParameter methodParam)` . Spring will consider this class to be the resolver if `supportsParameter` returns true , and will ignore it if it returns false. Notice that our method returns true when the method parameter is of type `UserAgent`.

With that out of the way,resolving the argument is just a matter of extracting out the header information from the underlying `HttpServletRequest`. 

Finally , let's register our `UserAgentAnnotationMethodResolver` with Spring.

```java
@Configuration
@EnableWebMvc
public class BindersConfig extends WebMvcConfigurerAdapter{
    @Override
     public void addArgumentResolvers(List<HandlerMethodArgumentResolver> argumentResolvers) {
         argumentResolvers.add( new UserAgentAnnotationMethodResolver());
     }
}
```
That's it ,people! As a proof of concept, let's fire up Spring and check url `http://localhost:8080/test` .

![localhost](https://cloud.githubusercontent.com/assets/7692552/19130948/123f895a-8b6b-11e6-80c3-2a25dec674a7.png "localhost")

You can download the sample [here](https://gitlab.com/ankushs92/spring-boot-sample-custom-resolver) .
