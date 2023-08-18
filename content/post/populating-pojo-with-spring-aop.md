+++
title = "Populating a POJO property with Spring AOP"
description = "Populating a POJO property with Spring AOP"

date = "2021-12-27"
tags = [
    "spring",
]
+++


An application can contain many cross-cutting concerns and Spring AOP is a good tool that allows you to add behaviors to a method without having to alter its implementation.

Imagine a simple example where you're developing an application that can provide blog posts to consumers, and they can filter the posts they want by their creation date. 

Note: The working example is accessible [here](https://github.com/Lemick/demo-spring-aop)

The model:
```java
@Data
@AllArgsConstructor
@NoArgsConstructor
public class BlogPost {

  private Instant creationDate;
  private String title;
}
```
The returned DTO:
```java 
@Data
@AllArgsConstructor
@NoArgsConstructor
@Builder
public class BlogPostsDTO {

  private List<BlogPost> blogPosts;
}
``` 
The controller endpoint (we declare the range query params explicitly here, but we can improve the reusability with an interceptor for example):
```java 
@GetMapping("/blogPosts")
public BlogPostsDTO findAll(
    @RequestParam("start") @NotNull Instant start,
    @RequestParam("end") @NotNull Instant end) {

  DateRange dateRange = new DateRange(start, end);
  List<BlogPost> blogPosts = blogPostDAO.findByCreationDate(dateRange);
  return BlogPostsDTO.builder().blogPosts(blogPosts).build();
}
```

The code is very simple, but the problem is that we will certainly need to return the range applied by the DAO every time the client passes a date filter, so treating it in this method is not a good approach.

It can also be imprecise to return the date range passed in the argument because we could not use this exact same date in the DAO (we may want to round the date to be able to **cache** the request for example).

We can solve this problem easily with Spring AOP. It allows you to add some behavior around methods. First, simply add an interface that declares that you can get and set a date filter on your response:
```java 
public interface IDateRangeFiltered {

  DateRange getDateRange();
  void setDateRange(DateRange dateRange);
}
```

Add the date range field and implement this interface in your DTO response:
```java 
@Data
@AllArgsConstructor
@NoArgsConstructor
@Builder
public class BlogPostsDTO implements IDateRangeFiltered {

  private DateRange dateRange;
  private List<BlogPost> blogPosts;
}
``` 

Finally, we add an AOP aspect that will be registered by Spring. We're recreating the date range from client arguments, but the date range can be registered in a request context for example, this aspect can inject dependencies like any other Spring Bean)
```java 
@Aspect
@Component
public class DateRangeSetterAspect {

  /**
   * The pointcut expression describes that this advice will be applied to:
   * classes WITHIN the package containing all the Spring Controllers
   * where the EXECUTED methods of these Controllers are returning a class which implements IDateRangeFiltered
   * finally, map the method ARGS by convenience
   */
  @Around("within(com.example.demo.controller.*) && " +
      "execution(public com.example.demo.model.contract.IDateRangeFiltered+ *(..)) &&" +
      "args(start, end)"
  )
  public Object setDateRange(ProceedingJoinPoint joinPoint, Instant start, Instant end) throws Throwable {

    DateRange appliedDateRange = new DateRange(start, end);
    IDateRangeFiltered returnedObject = ((IDateRangeFiltered) joinPoint.proceed());
    returnedObject.setDateRange(appliedDateRange);

    return returnedObject;
  }
}
``` 

We can now verify that our Aspect is correctly injected with a test:

```java
  @Test
  void _findAll_january_posts() {
    ResponseEntity<BlogPostsDTO> response = restTemplate.exchange(
        "http://localhost:" + port + "/blogPosts?start=2020-01-01T00:00:00Z&end=2020-01-30T23:59:59Z",
        HttpMethod.GET,
        null,
        BlogPostsDTO.class
    );

    BlogPostsDTO actual = response.getBody();

    assertEquals(2, actual.getBlogPosts().size(),
    "the two blog posts from January are returned");

    DateRange expectedDateRange = new DateRange(Instant.parse("2020-01-01T00:00:00Z"), Instant.parse("2020-01-30T23:59:59Z"));
    assertEquals(expectedDateRange, actual.getDateRange(), 
    "the date range applied is returned in the response");
  }
```


Now, every time you add a new endpoint that returns a filtered view, you can simply declare that your POJO implements the ```IDateRangeFiltered``` contract, and the response will be filled automatically.