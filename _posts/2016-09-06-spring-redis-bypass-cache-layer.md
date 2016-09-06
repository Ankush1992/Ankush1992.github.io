---
layout: post
title: Bypass Spring's Cache layer.
categories: libraries
tags: [groovy,spring-boot,caching,redis]
comments: true
---

Implementing a cache store like Redis or Memcached is a common scenario in web apps.Spring provides a great Caching abstraction layer with Redis as one of the implementations.
So say we are developing an app with an endpoint `developers/{developerId}` , which gives details about a particular developer with this `{developerId}` .We decide that once the result is fetched from the primary datastore(say MySQL),it is to be immediately cached in Redis.

So,we have a Developer class :

```groovy

@Entity
@Table(name="dev")
class Developer {
	
	@Id
	@GeneratedValue
	Integer id
	
	@Column
	String name
}
```

Also,a `DeveloperService` to interact with `DeveloperRepository`.


```groovy
@Service
@Transactional
class DeveloperService {
	
	@Autowired
	private DeveloperRepository developerRepository
	
	@Cacheable("devs")
	Developer findById(Integer id){
		developerRepository.findOne(id)
	}
}
```

```groovy
interface DeveloperRepository extends JpaRepository<Developer,Integer> {

}
```

Notice the `Cacheable("devs")` . If all goes well,the result will be fetched from MySQL and put into Redis. 
If the method `findById(Integer id)` is invoked again, the result will then be fetched from Redis (unless it is evicted somewhere using @CacheEvict).
So far ,so good ! 
<br />
But what if you run into an exception like this :
`org.springframework.data.redis.RedisConnectionFailureException: Cannot get Jedis connection; nested exception is redis.clients.jedis.exceptions.JedisConnectionException` 
<br />

You can't even make the connection with the Redis server,for god's sake ! Spring would just throw this exception,and the code inside the `findById(Integer id)` method will never be executed. 
This is not ideal behaviour.If something wrong happens with the Cache layer,you should immediately log the exception/error that took place, bypass it altogether, and execute the code inside the `findById(Integer id)` method so that everything functions as normal.

That is easy to do with Spring's [CacheErrorHandler](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/cache/interceptor/CacheErrorHandler.html) .

I'll show you how to do it,as well as do a minimal setup for Redis using Java Config.

```groovy
@EnableCaching
@Configuration
public class CacheConfig extends CachingConfigurerSupport {
	
	private static final Logger logger = LoggerFactory.getLogger(CacheConfig.class);

	
	@Bean
	public RedisTemplate<String,String> configureRedisTemplate(){
		final RedisTemplate<String,String> redisTemplate = new RedisTemplate<String,String>();
		redisTemplate.setConnectionFactory(configureRedisConnectionFactory());
		//JSON representation of Developer class will be stored in Redis. 
		redisTemplate.setValueSerializer(new Jackson2JsonRedisSerializer<Developer>(Developer.class));
		return redisTemplate;
	}
	
	@Bean
	public RedisConnectionFactory configureRedisConnectionFactory(){
		//Set of Defaults
		final JedisConnectionFactory jedisConnectionFactory = new JedisConnectionFactory();
		return jedisConnectionFactory;
	}
	
	@Bean
	public RedisCacheManager configureCacheManager(){
		final RedisCacheManager redisCacheManager = new RedisCacheManager(configureRedisTemplate());
		return redisCacheManager;
	}
	
	@Override
	public CacheErrorHandler errorHandler(){
			return new CacheErrorHandler(){

				@Override
				public void handleCacheClearError(RuntimeException ex, Cache cache) {
					//Do Certain stuff.
				}

				@Override
				public void handleCacheEvictError(RuntimeException ex, Cache cache, Object key) {
					//Do Certain stuff.					
				}

				@Override
				public void handleCacheGetError(RuntimeException ex, Cache cache, Object key) {
					logger.warn("Exception occured when fetching an item from Redis.Time : {} , Key : {} , Cache : {}, Exception : ",LocalDateTime.now(),key,cache.getName(),ex);
				}

				@Override
				public void handleCachePutError(RuntimeException ex, Cache cache, Object arg1, Object arg2) {
				     //Do Certain stuff.											
				}
		};
	}
}
```

All you need to do is to extend `CachingConfigurerSupport` and override `errorHandler()`.
Now, whenever you are performing a GET operation ,like our `findById(Integer id)` , and the same error occurs,then Spring would bypass the Cache layer and execute the method.
<br /> <br />
A `@Cacheable` maps to `handleCacheGetError(RuntimeException ex, Cache cache, Object key)` .
<br />
A `@CacheEvict` maps to `handleCacheEvictError(RuntimeException ex, Cache cache, Object key)` .
<br />
A `@CachePut` maps to `handleCachePutError(RuntimeException ex, Cache cache, Object arg1, Object arg2)`

You can use a similar strategy with other Cache backends that Spring provides.


You can download the sample [here](https://gitlab.com/ankushs92/sample-boot-redis-errorHandler).
