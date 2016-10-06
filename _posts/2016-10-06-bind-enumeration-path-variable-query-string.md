---
layout: post
title: Bind an enumeration to a @PathVariable or @RequestParam
categories: spring-boot
tags: [spring-boot]
comments: true
---

I am a big fan of Enumerations . Whenever you have to work with a String variable that can only hold a finite set of values , consider using an Enumeration.
Now,a `@PathVariable` or `@RequestParam` works fine when you need to resolve it to a String or an Integer or a Long.We will see how we can resolve a `@PathVariable` or `@RequestParam` to an enum.
We will work with a `Status` enum which has only two possible values.Our enum :

```java
public enum Status {
	ACTIVE,
	DEACTIVE;
	
	public static Status from(String val){
		if(!Strings.hasText(val)){
			return null;
		}
		val = val.toLowerCase();
		if(val.equals("active")){
			return Status.ACTIVE;
		}
		else if(val.equals("deactive")){
			return Status.DEACTIVE;
		}
		else{
			throw new IllegalArgumentException("Invalid value.Status can only be 'active' or 'deactive'");
		}
	}
}
```


Our end result looks like:

```java
@RestController
public class TestController {
	
	@GetMapping("test")
	public String getStatus(@RequestParam("status") final Status status){
		return "status is : " + status;
	}

}
```


We first write an implementation of  Spring's [PropertyEditorSupport](http://docs.oracle.com/javase/8/docs/api/java/beans/PropertyEditorSupport.html?is-external=true#setAsText-java.lang.String-) , and override method `setAsText` .

```java
public class StatusBinder  extends PropertyEditorSupport{
	
	@Override
    public void setAsText(final String text) throws IllegalArgumentException {
		if(Strings.hasText(text)){
			final Status status = Status.from(text);
			setValue(status);
		}
		else{
			setValue(null);
		}
    }
}
```


Now we just register our Binder with Spring's [WebDataBinder](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/bind/WebDataBinder.html) . 

```java

@RestController
public class TestController {

	@InitBinder
	void initBinder(final WebDataBinder binder){
		binder.registerCustomEditor(Status.class, new StatusBinder());
	}
	
	@GetMapping("test")
	public String getStatus(@RequestParam("status") final Status status){
		return "status is : " + status;
	}
}
```

Go on to `http://localhost:8080/test?status=active` .You would see :

![localhost](https://cloud.githubusercontent.com/assets/7692552/19141923/144a8cbc-8bb7-11e6-8c74-cb61547cdf94.png "localhost")

You can download the sample [here](https://gitlab.com/ankushs92/spring-boot-sample-enums-binder) .
