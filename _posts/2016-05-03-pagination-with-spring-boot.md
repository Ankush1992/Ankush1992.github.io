---
layout: post
title: Easy Pagination with Spring Boot.
categories: tutorial
tags: [spring-boot,pagination]
comments: true
---


In this short tutorial , we'll see how easy it is to set up Pagination in a Spring Boot app . 

## Setup
We use Spring Boot 1.3.3.RELEASE , with MySQL as the Database and Spring Data JPA abstraction to work with MySQL. Indeed ,it is the Spring Data JPA module
that makes it so easy to set up Pagination in a Spring boot app in the first place.

## Scenario

We expose an endpoint `/persons` . It will return a List of persons and other paging info(which we would see in a minute) based on the `page` and `size` parameters that were passed along with it.

For instance , ```/persons?page=0&size=3```  would return a batch of the first  3 persons from the database
```/persons?page=0&size=3`` would return the next batch . 

To begin with, we create a domain `Person` class . 

{% highlight groovy %}

@Entity
@Table
class Person {
	@Id
	@GeneratedValue
	Integer id
	
	@Column
	String name
	
	@Column
	Integer age
}


{% endhighlight %}

This is how our Controller looks like .

{% highlight groovy %}
@RestController
class PersonController {
	
	final PersonService personService
	
	@Autowired
	def PersonController( PersonService personService ){
		this.personService = personService
	}
	
	@RequestMapping(value="/persons",method=RequestMethod.GET)
	Page<Person> list( Pageable pageable){
		Page<Person> persons = personService.listAllByPage(pageable)
		persons
	} 
}
{% endhighlight %}

Notice that we haven't passed [RequestParams](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/bind/annotation/RequestParam.html) to our handler method . 
When the endpoint `/persons?page=0&size=3` is hit, Spring would automatically resolve the `page` and `size` parameters and create a `Pageable` instance .
We would then pass this `Pageable` instance to the Service layer ,which would pass it to our Repository layer .




Next,we create a `PersonRepository` class to interact with the database.

{% highlight groovy %}

interface PersonRepository extends PagingAndSortingRepository<Person,Integer> {

}

{% endhighlight %}

The PersonRepository class is just an interface. This might be weird for those coming from traditional Spring MVC world wherein you had to write implementation classes , and interacted with the database using Hibernate. Well, you don't need to do that anymore.
The PagingAndSortingRepository extends the [CrudRepository](http://docs.spring.io/spring-data/commons/docs/current/api/org/springframework/data/repository/CrudRepository.html) , thereby adding Paging capabilities .

Now all we need to do is to create a `PersonService` to expose the repository.

{% highlight groovy %}
@Service
@Transactional
class PersonServiceImpl implements PersonService {

	final PersonRepository personRepository
	
	@Autowired
	def PersonServiceImpl(PersonRepository personRepository){
		this.personRepository = personRepository
	}
	
	@Override
	 Page<Person> listAllByPage(Pageable pageable) {
		 personRepository.findAll(pageable)
	}
}
{% endhighlight %}



This is all, really ! Let's test it out . I created a test class to insert a batch of 20 Person objects . You can do this manually ofcourse.

{% highlight groovy %}
@RunWith(SpringJUnit4ClassRunner)
@SpringApplicationConfiguration(classes = SampleBootPaginationApplication)
@WebAppConfiguration
class SampleBootPaginationApplicationTests {
	
	@Autowired
	PersonRepository personRepository
	 
	@Test
	void contextLoads() { 
		def person 
		(1..20).each{
			person = new Person(name:"John $it", age : 22)
			personRepository.save(person)
		}
	}
}
{% endhighlight %}

Great,we're all done . Let's hit the endpoint with Postman .


![pagination_!](https://cloud.githubusercontent.com/assets/7692552/14977662/88e9a322-1133-11e6-9484-7d44b5b9882a.png "Pagination")


The snapshot doesn't display the entire JSON. So here it is .

{% highlight json %}
{
  "content": [
    {
      "id": 1,
      "name": "John 1",
      "age": 22
    },
    {
      "id": 2,
      "name": "John 2",
      "age": 22
    },
    {
      "id": 3,
      "name": "John 3",
      "age": 22
    }
  ],
  "last": false,
  "totalElements": 20,
  "totalPages": 7,
  "size": 3,
  "number": 0,
  "sort": null,
  "first": true,
  "numberOfElements": 3
}
{% endhighlight %}


You can download the sample [here](https://github.com/ankushs92/sample-boot-pagination)
