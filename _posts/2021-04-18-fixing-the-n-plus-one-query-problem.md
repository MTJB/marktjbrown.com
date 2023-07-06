---
layout: post
title:  Fixing the N+1 query problem
description: In an earlier blog post, I touched upon the cardinal sin of performance issues - N+1 queries. In this b...
date:   2021-04-05 15:01:35 +0300
image:  '/images/posts/2021-04-18-fixing-the-n-plus-one-query-problem/nplusone.png'
tags:   [hibernate, java, sql]
---

<blockquote class="twitter-tweet tw-align-center"><p lang="en" dir="ltr">Everybody has a testing environment. Some people are lucky enough enough to have a totally separate environment to run production in.</p>&mdash; stahnma (@stahnma) <a href="https://twitter.com/stahnma/status/634849376343429120?ref_src=twsrc%5Etfw">August 21, 2015</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

## 👋 Introduction
In an earlier [blog post](https://marktjbrown.com/improve-query-caching-with-in-clause-padding), I touched upon the cardinal sin of performance issues - N+1 queries. In this blog post I am going to explain through a few examples what the N+1 query problem is, and also suggest how to fix each in your application code.

The N+1 query problem is sometimes hard to diagnose from reading your code - which is another reason why [you should check the generated sql](https://marktjbrown.com/criteria-query) from your application. With the N+1 problem, you will often find that the issue is not the performance of the query generated, but instead the sum of number of executions and that query time. Therefore, the larger the value of N, the more queries that will be executed, and the larger the performance impact. Unless you test your application at scale, you may find that you do not flush out some of these issues until much later in the development cycle..

## 🧐 Examples
Consider the following `car_garage` and `car` tables which from a _one-to-many_ relationship;

![Schema]({{site.baseurl}}/images/posts/2021-04-18-fixing-the-n-plus-one-query-problem/schema.png)

#### FetchType.EAGER
Using `FetchType.EAGER` is generally best avoided because you are going to fetch more data than is required, but also because this fetch type is prone to N+1 issues. The good news, however, this is not always used by default but if your entity relationship is using `@ManyToOne` or `@OneToOne` then unfortunately this is used the default value.

So if your mappings look like this;

```java
@ManyToOne
@JoinColumn(name = "garage_id", nullable = false)
private CarGarage garage;
```

Then you are using the `FetchType.EAGER` strategy, and every time you fetch the `car` entity;

```java
@Query("SELECT c FROM Car c")
List<Car> findAllCars();
```

Then you are going to trigger the N+1 issue;

```sql
select car0_.id as id1_0_, 
       car0_.garage_id as garage_i5_0_, 
       car0_.make as make2_0_, 
       car0_.model as model3_0_, 
       car0_.registration as registra4_0_ 
from car car0_

select cargarage0_.id as id1_1_0_, cargarage0_.name as name2_1_0_ from car_garage cargarage0_ where cargarage0_.id=?
select cargarage0_.id as id1_1_0_, cargarage0_.name as name2_1_0_ from car_garage cargarage0_ where cargarage0_.id=?
select cargarage0_.id as id1_1_0_, cargarage0_.name as name2_1_0_ from car_garage cargarage0_ where cargarage0_.id=?
select cargarage0_.id as id1_1_0_, cargarage0_.name as name2_1_0_ from car_garage cargarage0_ where cargarage0_.id=?
```

Notice how the additional `SELECT` statements are executed too because the `car_garage` association has to be fetched after loading the list of `car` entities.

#### FetchType.LAZY
Almost contradictory to the above example, switching to `FetchType.LAZY` can also cause the N+1 query problem, for example if switching the `CarGarage` association to;

```java
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "garage_id", nullable = false)
private CarGarage garage;
```
Fetching a list of car entities will now only execute one sql statement, but when referencing the lazy-loaded association;

```java
List<Car> cars = (List<Car>) carRepository.findAllCars();
Set<String> garageNames = cars.stream()
        .map(Car::getGarage)
        .map(CarGarage::getName)
        .collect(Collectors.toSet());
```

You will again trigger the N+1 query problem;

```sql
select cargarage0_.id as id1_1_0_, cargarage0_.name as name2_1_0_ from car_garage cargarage0_ where cargarage0_.id=?
select cargarage0_.id as id1_1_0_, cargarage0_.name as name2_1_0_ from car_garage cargarage0_ where cargarage0_.id=?
select cargarage0_.id as id1_1_0_, cargarage0_.name as name2_1_0_ from car_garage cargarage0_ where cargarage0_.id=?
select cargarage0_.id as id1_1_0_, cargarage0_.name as name2_1_0_ from car_garage cargarage0_ where cargarage0_.id=?
```

## 🛠 Fixing the N+1 issue
The first step you should take to fix the N+1 issue should be to ask yourself; "Do I even need to load this data?"

If the answer is no, you should ensure that your `@ManyToOne` (or `@OneToMany`) association has its `fetchType` set to `FetchType.LAZY` - in fact, it is better to use this by default to avoid the N+1 issue creeping in across all of your entity relationships.

If you do, there are a few ways of solving this, and it depends on a number of things - including inspecting the execution plan used for any of these options and choosing the one that is most optimal for your example, while also weighing up the pros and cons of each solution.

#### Using JOIN FETCH
By using `JOIN FETCH` we can force the generated query to use an `INNER JOIN` to load the `car_garage` rows from the database, and therefore eradicating the N+1 issue;

```java
@Query("SELECT c FROM Car c JOIN FETCH c.garage")
List<Car> findAllCars();
```

Will generate the follow query;

```sql
select car0_.id as id1_0_0_, 
       cargarage1_.id as id1_1_1_, 
       car0_.garage_id as garage_i5_0_0_, 
       car0_.make as make2_0_0_, 
       car0_.model as model3_0_0_, 
       car0_.registration as registra4_0_0_, 
       cargarage1_.name as name2_1_1_ 
from car car0_ 
inner join car_garage cargarage1_ on car0_.garage_id=cargarage1_.id 
```

##### Entity Graphs
Entity graphs are another way of loading the garage association and avoiding the N+1 issue, I will cover these in more detail in a later post.

By specifying a new entity graph for the `garage` relationship.

```java
@NamedEntityGraph(
    name = "Car.garage",
    attributeNodes = { @NamedAttributeNode(value = "garage") }
)
public class Car {

@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "garage_id", nullable = false)
private CarGarage garage;

// ETC
```

And updating the query to make use of this entity graph.

```java
@EntityGraph(value = "Car.garage", type = EntityGraph.EntityGraphType.LOAD)
List<Car> findAll();
```

Then the following query is generated;

```sql
select car0_.id as id1_0_0_, 
       cargarage1_.id as id1_1_1_, 
       car0_.garage_id as garage_i5_0_0_, 
       car0_.make as make2_0_0_, 
       car0_.model as model3_0_0_, 
       car0_.registration as registra4_0_0_, 
       cargarage1_.name as name2_1_1_ 
from car car0_ 
left outer join car_garage cargarage1_ on car0_.garage_id=cargarage1_.id
```

## ⚡️ Conclusion
The N+1 can be subtle, and often undiagnosed for some time - it is best to do your best to avoid it by always defaulting to `FetchType.LAZY` in your entity relationships, this way you have to adjust your code to fetch the extra information - rather than always having it there when you don't need it to be!