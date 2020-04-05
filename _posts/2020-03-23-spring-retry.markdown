---
layout: post
title:  "Spring Retry Aspect Oriented Programming"
date:   2019-03-22 21:00:00 -0400
categories: design
---

# Introduction
Let's say you have a microservices architecture, where the failure
of an unknown system can cause the failure of your system, paraphrased
from Leslie Lamport. This means that errors, or bad responses
from other systems are a common occurrence, relatively speaking.

This means each microservice must be fault tolerant, and able to handle
bad states or fail gracefully. Spring framework can facilitate this
nearly effortlessly, and the point of this post is an introduction
to automating retry logic. This is useful for distributed systems
whose failures may be successes on subsequent requests, whether it be
HTTP or some other protocol.

This assumes some familiarity with Spring, especially `@Service`.

## The Problem
Your system depends on some other brittle system, who fails at some
unknown likelihood per request (hopefully low!). You know that if you retry
a request, it could succeed on a second, or even third attempt.
Spring framework handles this easily. Enter `spring-retry`.

## Spring Retry
Spring retry is exposed through 
[Aspect Oriented Programming](https://docs.spring.io/spring/docs/2.5.x/reference/aop.html).
Aspect Oriented Programming is essentially a natural
extension of inversion of control concepts, where logic can be
applied to some service, *independent* of the actual business logic.
More specifically, retry logic can be applied to a Spring `@Service`.
This means that adding an annotation to a Spring `@Service` is almost
all that is needed to have graceful retries.

### Dependencies
```
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-aspects</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.retry</groupId>
    <artifactId>spring-retry</artifactId>
</dependency>
```
`spring-aspects` is needed because `spring-retry` is made available
through proxying facilitated by `spring-aspects`.

### Service Level Implementation
```
@Service
public class ReliableResource {

    public String getResource(){
        return "Resource";
    }
}
```
The above is an example of your typical Spring `@Service` that retrieves
some resource.

```
@Service
public class BrittleResource {

    public String getBrittleResource() throws RetryableException{
        int result = new Random().nextInt() % 2;
        if(result == 0){
            System.out.println("Failed to retrieve data!");
            throw new RetryableException("Brittle service failure");
        }
        return "Brittle resource";
    }
}
```
The above is a brittle service that fails exactly half the time.
It would be nice to have retries for this service. Note that `RetryableException`
is a class I defined, indicating that the specific situation that arises
at the `throw` of this exception is able to be retried. 

There may
be other cases that are not retryable. It is an anti-pattern to retry
on all `exception`, because there may be exceptions caused by developer
error in our system, and we should not add unnecessary load to
other systems. Retryable situations should be identified,
and the `throw new RetryableException(...)` is a case
that should be retried, such as a a timeout error, or a `500` response
back from a server. An example of a situation that should not be retried
would be `400`, malformed request, or specific database `SQLException`,
possibly a malformed query.

```
@Retryable(include = RetryableException.class)
public String getBrittleResource() throws RetryableException{
    int result = new Random().nextInt() % 2;
    if(result == 0){
        System.out.println("Failed to retrieve data!");
        throw new RetryableException("Brittle service failure");
    }
    return "Brittle resource";
}
```
`include = RetryableException.class` indicates that *only* thrown
`RetryableException` exceptions should be retried. The default retry
behavior is a maximum of 3 attempts with 1 second between attempts.
There are various backoff schemes, including exponential. That configuration is
achieved in our `@Configuration` class.


### Configuration
```
@Configuration
@ComponentScan(basePackages = "com.cmorterud.examples.spring.retryTemplateExample")
@EnableRetry
public class AppConfig {

    @Bean
    public RetryTemplate retryTemplate() {
        RetryTemplate retryTemplate = new RetryTemplate();

        FixedBackOffPolicy fixedBackOffPolicy = new FixedBackOffPolicy();
        fixedBackOffPolicy.setBackOffPeriod(500);
        retryTemplate.setBackOffPolicy(fixedBackOffPolicy);

        SimpleRetryPolicy retryPolicy = new SimpleRetryPolicy();
        retryPolicy.setMaxAttempts(3);
        retryTemplate.setRetryPolicy(retryPolicy);

        return retryTemplate;
    }
}
```
In our Spring `@Configuration` class, we need to `@EnableRetry`
so that the Spring-level class proxy provided
by `spring-aspects` should handle retry logic.
`@ComponentScan` is basic Spring configuration, more at 
[Baeldung](https://www.baeldung.com/spring-component-scanning).

The `@Bean` is to override the definition for a `RetryTemplate` object
that governs retry logic within the Spring-level class proxy.

### Controller
The `@Controller` is nearly abstract with respect to the service level
implementation for the brittle `@Service` dependency. The controller
only needs to handle the exception thrown by the brittle service,
and return the appropriate response to the client.

```
@Controller
@EnableAutoConfiguration
@RequestMapping("/resource")
public class ExampleController {
    @Autowired
    private ReliableResource reliableResource;

    @Autowired
    private BrittleResource brittleResource;

    @RequestMapping("/getResource")
    ResponseEntity<String> getResource(){
        return new ResponseEntity<String>(reliableResource.getResource(), HttpStatus.OK);
    }

    @RequestMapping("/getBrittleResource")
    ResponseEntity<String> getBrittle() {
        String brittleResourceData;

        try {
            brittleResourceData = brittleResource.getBrittleResource();
        } catch (RetryableException e) {
            brittleResourceData = null;
        }

        if(brittleResourceData == null) {
            return new ResponseEntity<>("failed to retrieve data!", HttpStatus.INTERNAL_SERVER_ERROR);
        }
        return new ResponseEntity<>(brittleResourceData, HttpStatus.OK);
    }
}
```

## Testing
I wrote some tests using `MockMvc` that validate the retries occurring.
```
@SpringBootTest
@AutoConfigureMockMvc
class RetryTemplateExampleApplicationTests {

	@Autowired
	private MockMvc mockMvc;

	@Test
	void reliableResourceSucceeds() throws Exception {
		this.mockMvc.perform(get("/resource/getResource")).andDo(print()).andExpect(status().isOk())
				.andExpect(content().string("Resource"));
	}

	@Test
	void brittleResourceSucceeds() throws Exception {
		this.mockMvc.perform(get("/resource/getBrittleResource")).andDo(print()).andExpect(status().isOk())
				.andExpect(content().string("Brittle resource"));
	}
}
```
I can see two failures (and success on the third) when running the tests.
```
Failed to retrieve data!
Failed to retrieve data!
```

## [Repository](https://github.com/cmorterud/SpringRetryExample)

## Caveat
Spring-level class proxying _*only*_ works if the `Retryable` method
is called from _*outside*_ the class. Meaning, if I had a definition
of the brittle resource like this,

```
public String getBrittleResource() throws RetryableException {
    return getActualBrittleResource();
}

@Retryable(include = RetryableException.class)
private String getActualBrittleResource() throws RetryableException {
    int result = new Random().nextInt() % 2;
    if(result == 0){
        System.out.println("Failed to retrieve data!");
        throw new RetryableException("Brittle service failure");
    }
    return "Brittle resource";
}
```
when `RetryableException` is thrown, no retries will occur!

## Thanks!
I learned most of this through trial and error as well as
[Baeldung's article](https://www.baeldung.com/spring-retry).
You can find the working `maven` project at
[my github repository](https://github.com/cmorterud/SpringRetryExample) and
please feel free to email me with any questions or concerns. 
Thanks for reading!
