---
layout: post
title:  Using Testcontainers for Integration Testing
description: As a follow on from my last post about running sql server in docker, I thought i’d write about something...
date:   2020-10-20 15:01:35 +0300
image:  '/images/posts/2020-10-31-improve-query-caching-with-in-clause-padding/padding.jpg'
tags:   [hibernate, java, sql]
---

## 👋 Introduction
In my experience, the number one cause of application performance problems is not your application code – it’s your persistence layer. Problems in this area can be caused by many different ‘sins’; Improper entity relationships (think LAZY vs EAGER fetching), Inefficient queries or indeed the cardinal sin – N+1 queries! A lot of the time the application can end up in this way as a result of a lack of awareness of what you are asking the persistence layer to do. Often, you’ll find that simply enabling logging of sql statements will open your eyes to the problem. In fact, when using a data access framework that generates statements on your behalf – it should be *mandatory* that you inspect the generated statements to ensure both their effectiveness, and their performance.

Today we’re going to talk about a simple optimisation trick that comes after you have done all you can do for the query, it’s been carefully developed, tested, and you have given consideration to query optimisation – now what? Well you can enable _IN clause padding_ to get the query cache on your side so that less execution plans will be required for your query.

All code examples in this post are available on my [GitHub](https://github.com/MTJB/blog-code-examples).

## 🤨 The Problem
#### Execution plans
Execution plans (or query plans) are built by the query optimiser and is an attempt to calculate the most efficient way to process the request (the SQL query). SQL Server has to build an execution plan for each request that it has to execute. The execution plan is build based on several considerations;

The tables it needs to join.
The Indexes to use
The sub-queries it has to execute.
How aggregations of `GROUP BY` are calculated.
The estimated cost and load the operations place on the system.
Other even more complex considerations.
Based on combinations of the above, it cannot always be guaranteed that ‘best’ execution plan will be chosen – For example if your query is not making use of correct filtering, the optimiser may decide not to use any indexes to fulfil the query results. Therefore, it is imperative that you are producing carefully considered queries.

SQL has to put a lot of work to build an execution plan, so it caches the execution plan in memory to avoid having to do the same work over and over again.

#### The query cache
SQL Server uses the Query Cache to reuse plans. SQL Server can avoid the overhead of calculating the execution plan for each request and speed up the execution of the queries. The query cache allows SQL Server to reuse Execution Plans for subsequent requests. Within the query cache, additional information is captured, for example the number of times a query has been executed, resources used, etc, so inspecting the contents of the query cache can be a way of diagnosing performance issues in your query statements.

The query cache can be cleared by executing T-SQL commands, but this is not recommended in production code.

#### Default behaviour
Consider the following entity:

```java
@Entity
public class Customer {
 
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
 
    private String firstName;
 
    private String lastName;
 
    // Getters/Setters omitted
}
```

Now imagine you want to load multiple `Customer` entities by `id` , so you have written the following query:

```java
@Query("FROM Customer WHERE id IN :ids")
Set<Customer> findAllById(@Param("ids") Collection<Long> ids);
```

When running the follow test cases:

```groovy
def "Find all by id"(Set<Long> ids) {
        expect: "Size to be equal"
            customerRepository.findAllById(ids).size() == ids.size()
        where:
            ids                                 | _
            [1L, 2L]                            | _
            [2L, 3L]                            | _
            [3L, 4L, 5L]                        | _
            [1L, 2L, 3L, 4L, 5L]                | _
            [4L, 5L, 6L, 7L, 8L, 9L]            | _
            [4L, 5L, 6L, 7L, 8L, 9L, 10L]       | _
            [3L, 4L, 5L, 6L, 7L, 8L, 9L, 10L]   | _
}
```

The following SQL statements are executed by Hibernate:

```
select customer0_.id as id1_1_, customer0_.first_name as first_na2_1_, customer0_.last_name as last_nam3_1_ from customer customer0_ where customer0_.id in (? , ?)
select customer0_.id as id1_1_, customer0_.first_name as first_na2_1_, customer0_.last_name as last_nam3_1_ from customer customer0_ where customer0_.id in (? , ?)
select customer0_.id as id1_1_, customer0_.first_name as first_na2_1_, customer0_.last_name as last_nam3_1_ from customer customer0_ where customer0_.id in (? , ? , ?)
select customer0_.id as id1_1_, customer0_.first_name as first_na2_1_, customer0_.last_name as last_nam3_1_ from customer customer0_ where customer0_.id in (? , ? , ? , ? , ?)
select customer0_.id as id1_1_, customer0_.first_name as first_na2_1_, customer0_.last_name as last_nam3_1_ from customer customer0_ where customer0_.id in (? , ? , ? , ? , ? , ?)
select customer0_.id as id1_1_, customer0_.first_name as first_na2_1_, customer0_.last_name as last_nam3_1_ from customer customer0_ where customer0_.id in (? , ? , ? , ? , ? , ? , ?)
select customer0_.id as id1_1_, customer0_.first_name as first_na2_1_, customer0_.last_name as last_nam3_1_ from customer customer0_ where customer0_.id in (? , ? , ? , ? , ? , ? , ? , ?)

```

The query was executed 7 times in total in the test, only 2 of which will be sharing an execution plan as the other 5 queries had a differing number of bind parameters.

## 🧐 Enabling IN clause padding
If you enable the ‘padding’ property in Hibernate; `hibernate.query.in_clause_parameter_padding`, e.g;

```
spring.jpa.properties.hibernate.query.in_clause_parameter_padding=true
```

And re-running the test cases, the following statements are executed.

```
select customer0_.id as id1_1_, customer0_.first_name as first_na2_1_, customer0_.last_name as last_nam3_1_ from customer customer0_ where customer0_.id in (? , ?)
select customer0_.id as id1_1_, customer0_.first_name as first_na2_1_, customer0_.last_name as last_nam3_1_ from customer customer0_ where customer0_.id in (? , ?)
select customer0_.id as id1_1_, customer0_.first_name as first_na2_1_, customer0_.last_name as last_nam3_1_ from customer customer0_ where customer0_.id in (? , ? , ? , ?)
select customer0_.id as id1_1_, customer0_.first_name as first_na2_1_, customer0_.last_name as last_nam3_1_ from customer customer0_ where customer0_.id in (? , ? , ? , ? , ? , ? , ? , ?)
select customer0_.id as id1_1_, customer0_.first_name as first_na2_1_, customer0_.last_name as last_nam3_1_ from customer customer0_ where customer0_.id in (? , ? , ? , ? , ? , ? , ? , ?)
select customer0_.id as id1_1_, customer0_.first_name as first_na2_1_, customer0_.last_name as last_nam3_1_ from customer customer0_ where customer0_.id in (? , ? , ? , ? , ? , ? , ? , ?)
select customer0_.id as id1_1_, customer0_.first_name as first_na2_1_, customer0_.last_name as last_nam3_1_ from customer customer0_ where customer0_.id in (? , ? , ? , ? , ? , ? , ? , ?)
```

This time, only 3 execution plans are required – the statements being executed are using either 2, 4, or 8 bind parameters.

This is possible because Hibernate is now padding parameters to the next power of two. Test cases 4-7 in my example were very deliberately outlined to exaggerate this problem – each of those can now use just one query statement with 8 bind parameters, previously this was 4 separate query statements with [5, 6, 7, 8](https://www.youtube.com/watch?v=4NO-h9PFum4&ab_channel=StepsVEVO) parameters respectively.

## 🔥 Conclusion
As you can see, this one-liner is an easy way to boost your application’s statement caching potential – free performance gains!

So what’s the downside? Well, JDBC has a limit on the amount of parameters that can be bound – 2100 to be precise – so you need to be careful that rounding does not exceed this. Honestly though, if your queries _do_ bind that many parameters, then you should probably also have a look at those too as that will be another source of performance issues.