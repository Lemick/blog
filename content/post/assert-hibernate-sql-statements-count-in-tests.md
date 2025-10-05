+++
title = "How to assert Hibernate SQL statement count in tests"

date = "2022-04-24"
tags = [
    "spring",
    "jpa",
]
+++

The data-access layer is often where performance problems emerge and what defines your application response time. Uncontrolled SQL queries can produce long-running transactions that affect the scalability of your software.
Hibernate is a powerful tool, but it's often criticized because it can sometimes have unpredictable behavior when it converts all your entities fetch/update to SQL statements (N+1 Select, Cartesian product…). 

## How to track Hibernate SQL statements

The easiest way to track the SQL statements is by enabling their logging, unfortunately, even if it's useful in development, that cannot prevent regression of your data access queries in production, we need some testing to help us.
Spring tests are useful for building integration tests. They are often used with an in-memory H2 database, which allows quicker tests that approach a production-like relational database.

## Hibernate-Query-Utils

[This tool I developed](https://github.com/Lemick/hibernate-query-asserts) has the advantage of being simple to integrate with, it consists of a Hibernate SQL Inspector and a Spring Test Listener that will parse and count the SQL queries executed by Hibernate. Different statement types (Select, Update, Insert, Delete) can be counted in your test method by adding an annotation.

Let's demonstrate it! I will use a ```BlogPost``` and a ```PostComment``` entity. The full working example with its tests [can be found here](https://github.com/Lemick/demo-hibernate-query-utils).

```java
@Entity
@Data
@NoArgsConstructor
public class BlogPost {

@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;

@OneToMany(fetch = FetchType.LAZY, mappedBy = "blogPost",
           cascade = CascadeType.ALL, orphanRemoval = true)
private List<PostComment> postComments;

private String title;

public BlogPost(String title) {
    this.postComments = new ArrayList<>();
    this.title = title;
}

public void addComment(PostComment postComment) {
    postComment.setBlogPost(this);
    this.postComments.add(postComment);
}
}
```

```java
@Entity
@Data
@NoArgsConstructor
public class PostComment {

@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;

@ManyToOne(fetch = FetchType.LAZY)
BlogPost blogPost;

private String content;

public PostComment(String content) {
        this.content = content;
    }
}
```

## Detecting inserts

The following integration test will verify that 6 SQL Insert statements are going to be generated. This annotation will also ensure that 0 Update, 0 Select and 0 Delete are executed since we specified nothing:
```java
@Test
@Transactional
@Commit
@AssertHibernateSQLCount(inserts = 6)
void create_three_blog_posts() {
    BlogPost post_1 = new BlogPost("Blog post 1");
    post_1.addComment(new PostComment("Good article"));
    blogPostRepository.save(post_1);

    BlogPost post_2 = new BlogPost("Blog post 2");
    post_2.addComment(new PostComment("Nice"));
    blogPostRepository.save(post_2);

    BlogPost post_3 = new BlogPost("Blog post 3");
    post_3.addComment(new PostComment("Coooool"));
    blogPostRepository.save(post_3);
}
```

You can guess that these inserts are not batched in my case, if the assertion *is not valid*, a message containing the executed SQL statements is shown:

```
Expected 3 INSERT but got 6:
     => '/* insert com.lemick.testdatasourceproxy.entity.BlogPost */ insert into blog_post (id, title) values (default, ?)'
     => '/* insert com.lemick.testdatasourceproxy.entity.PostComment */ insert into post_comment (id, blog_post_id, content) values (default, ?, ?)'
     => '/* insert com.lemick.testdatasourceproxy.entity.BlogPost */ insert into blog_post (id, title) values (default, ?)'
     => '/* insert com.lemick.testdatasourceproxy.entity.PostComment */ insert into post_comment (id, blog_post_id, content) values (default, ?, ?)'
     => '/* insert com.lemick.testdatasourceproxy.entity.BlogPost */ insert into blog_post (id, title) values (default, ?)'
     => '/* insert com.lemick.testdatasourceproxy.entity.PostComment */ insert into post_comment (id, blog_post_id, content) values (default, ?, ?)'
```

## Detecting N+1 SELECT

N+1 Select (which is a 1+N Select in reality), is a common problem that often comes from fetching manually lazy childs associations, a simple test can detect that all blog posts and their comments are fetched **in one Select**, preventing huge regressions:
```java
@Test
@Transactional
@AssertHibernateSQLCount(selects = 1)  // <= This will warn you if you're triggering N+1 SELECT
void fetch_post_and_comments_with_one_select() {
    blogPostRepository.findBlogPostWithComments().forEach(blogPost ->
            assertEquals(1, blogPost.getPostComments().size(), 
            "all blog posts have one comment")
    );
}
```

## HTTP integration test

It's not mandatory for your test to be transactional, counting SQL queries within HTTP integration tests is supported. Here we verify that our HTTP POST generated 1 SQL Insert:
```java
@Test
@AssertHibernateSQLCount(inserts = 1)
void create_one_entity_from_endpoint() {
    BlogPost requestBody = new BlogPost("My new blog post");

    BlogPost responseBody = restTemplate.postForObject("/blogPosts", requestBody, BlogPost.class);

    assertEquals("My new blog post", responseBody.getTitle(), 
    "The blog post created is returned");
}
```

## Comparison with JDBC Datasource-Proxy

I've found that all the existing implementations of Hibernate SQL statements assertions are based on [datasource-proxy](https://github.com/ttddyy/datasource-proxy), that add a proxy around your Datasource for controlling or inspecting its statements, it's a powerful tool. But that inspired me to try to provide my built-in solution for Spring Test and that is more natural to Hibernate write-behind mechanism.
For example, the assert-tooling coming with datasource-proxy lib is based on static calls, which can lead to a test like this:

```java
@Test
@Transactional
void _test_with_datasource_proxy() {
    BlogPost post = blogPostRepository.findById(1L);
    post.setTitle("New title for blog post 1");

    entityManager.flush(); 
    assertUpdateStatements(1L);
}
```

The drawback is that you need to flush explicitly since Hibernate's write-behind mechanism will try to defer the execution of the update event to the end of your transaction, which is unfortunately out of your reach, after the end of your test method. This drawback is minimized if you're only testing methods that are already wrapped with ```@Transactional``` though.

For this reason, I found that it was more relevant to use an annotation that wraps the execution of the tests, allowing us to flush just before the end of the test transaction, and to do validate the assertion at that moment.  But I think that depends on how you structure your tests.
On the other side, the drawback of my solution is that it is intentionally coupled with Hibernate and Spring, compared to datasource-proxy that is based on the JDBC API.

## Conclusion
I hope it was clear, this was pretty fun to code. I surely will try to add some tooling that can be useful in this test context, like integrating assertions for L2C caching hits, for example.
