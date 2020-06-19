---
layout: post
title:  "Complex Function Anti Pattern"
date:   2020-06-20 21:00:00 -0400
categories: design
---

# Introduction
You're writing a function. When should you break the function up?
Is this function too long? Who cares? This may be a little bit of a subjective
topic, but in my junior opinion, a reasonable and historical guidepost
is the [Linux kernel style guide](https://www.kernel.org/doc/html/v4.10/process/coding-style.html#functions).

## The Details
### 1. One single purpose
A function should do only one thing, and do it well. Here's an example of
a function with more than one purpose.
```java
void retrieveFizzBuzz(String number) {
    RestTemplate template = new RestTemplate(FIZZ_BUZZ_SERVER_URL);
    try {
        HttpResponse httpResponse = template.post(number);
        String fizzBuzzStatus = httpResponse.body();
        if(fizzBuzzStatus == null || fizzBuzzStatus.isEmpty()) {
            throw new RuntimeException(String.format("number %s is not fizzbuzz!!", number));
        }
        System.out.println(String.format("%s is %s", number, fizzBuzzStatus));
    } catch (Exception e) {
        System.out.println(String.format("Exception occurred when retrieveFizzBuzz: %s", e.getMessage(), e);
    }
}
```
This function `POST`s a resource, retrieves and parses the response, and then
validates the response (with respect to business logic), and performs
the business logic (what we are really interested in).

We can break this function into two shorter functions. This specific
example may be a bit pedantic but otherwise illustrates the idea.

```java
public class FizzBuzzService {
    private static String FIZZ_BUZZ_SERVER_URL = "http://localhost:8080/fizzbuzz";

    private String retrieveFizzBuzz(String number) throws Exception {
        RestTemplate template = new RestTemplate(FIZZ_BUZZ_SERVER_URL);
        try {
            return template.post(number).body();
        } catch (Exception e) {
            System.out.println(String.format("Exception occurred when retrieveFizzBuzz: %s", e.getMessage(), e);
            throw e;
        }
    }

    private void performBusinessLogic(String fizzBuzzResult) throws RuntimeException {
        if(fizzBuzzStatus == null || fizzBuzzStatus.isEmpty()) {
            throw new RuntimeException(String.format("number %s is not fizzbuzz!!", number));
        }
        System.out.println(String.format("%s is %s", number, fizzBuzzStatus));
    }

    public void determineFizzBuzz(String number) {
        try {
            String fizzBuzzStatus = retrieveFizzBuzz(number);
            performBusinessLogic(fizzBuzzStatus);    
        } catch (Exception e) {
            // do whatever we need to do in the context of business logic
            // in error scenarios
        }
    }
}
```

Why does this matter? Each function can be independently tested and validated
with respect to expected behavior, and requires less logical deduction
to understand the output of a given function. This is a pet example,
but much more convoluted functions can be difficult to write
satisfactory tests for, and hard to understand.


### 2. 24 line maximum function length
This is arguably a proxy of the single purpose principle, because
if a function is over 24 lines, then it may be trying to accomplish more
than one goal. But, in general, short function lengths are much more readable.

### 3. Maximum indent level of 3
This is an interesting one. Often, you can invert `if` semantics to reduce
the amount of nesting. Multiple branches (`if` conditions) of logic
could provide hints that a given block of code has more than a single responsibility.

```java
public class FizzBuzzService {
    private static String FIZZ_BUZZ_SERVER_URL = "http://localhost:8080/fizzbuzz";

    String retrieveFizzBuzz(String number) {
        if(number != null && !number.isEmpty()) {
            RestTemplate template = new RestTemplate(FIZZ_BUZZ_SERVER_URL);
            try {
                String fizzBuzzStatus = template.post(number).body();
                if(fizzBuzzStatus != null && !fizzBuzzStatus.isEmpty()) {
                    System.out.println(fizzBuzzStatus);
                } else {
                    System.out.println("Empty fizzbuzz response!");
                }
            } catch (Exception e) {
                System.out.println(String.format("Exception occurred when retrieveFizzBuzz: %s", e.getMessage(), e);
                throw e;
            }
        } else {
            throw new RuntimeException("number is invalid!");
        }
    }
}
```
The above code has three levels of nesting, one provided by the `try` block.
Here's how we can refactor it.
```java
public class FizzBuzzService {
    private static String FIZZ_BUZZ_SERVER_URL = "http://localhost:8080/fizzbuzz";

    String retrieveFizzBuzz(String number) {
        if(number == null || number.isEmpty()) {
            throw new RuntimeException("number is invalid!");
        }
        RestTemplate template = new RestTemplate(FIZZ_BUZZ_SERVER_URL);
        try {
            String fizzBuzzStatus = template.post(number).body();
            if(fizzBuzzStatus == null || fizzBuzzStatus.isEmpty()) {
                System.out.println("Empty fizzbuzz response!");
                return;
            }
            System.out.println(fizzBuzzStatus);
        } catch (Exception e) {
            System.out.println(String.format("Exception occurred when retrieveFizzBuzz: %s", e.getMessage(), e);
            throw e;
        }
    }
}
```
In general, one can invert `if` semantics in a productive manner
by identifying the control flow. Is there a scenario where we need 
to handle an error? Immediately check for the error and `throw`, as opposed
to validating data, and if it is valid, proceed with the rest of the logic.
Make `if` conditions check for errors, and if they are present, `throw`,
eliminating a level of nesting.


## Thanks!
Please feel free to email me at [{{ site.email }}](mailto:{{ site.email }})
with any questions or concerns. 
Thanks for reading!
