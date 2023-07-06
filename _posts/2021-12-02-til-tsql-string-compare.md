---
layout: post
title:  TIL - Comparing Strings with trailing spaces using TSQL
description: I set off down this path of enlightenment as I investigated a customer issue that “could never happen”...
date:   2021-12-02 15:01:35 +0300
image:  '/images/posts/2021-12-02-til-tsql-string-compare/hero.png'
tags:   [sql]
---

---
title: TIL - Comparing Strings with trailing spaces using TSQL
author: Mark Brown
date: 2021-12-02
hero: ./images/hero.png
slug: /til-tsql-string-compare
excerpt: I set off down this path of enlightenment as I investigated a customer issue that “could never happen”.
---

## Summary 📖
I set off down this path of enlightenment as I investigated a customer issue that "could never happen". I paused briefly to wipe the egg off my face then dug into how it could happen.

To keep my Car Garage analogy flowing, imagine you own a franchise that sells many flavours of car - BMW, Mercedes, and the like. The used car market is booming and you now automate the process of diverting incoming shipments to the right garage - for example you want all Mercedes to go to Garage A, All BMWs to Garage B, and so on (I'm really trying here..).

Now imagine you receive a shipment of fancy new Mercedes', and you make a typo inputting those and instead input "Mercedes " - to your shock these cars are not assigned to the right place 😱 🤯.

The Java developer in me looks at the 2 strings and thinks "But of course, they don't match!". To which the SQL developer in me now says - ah of course, ANSI/ISO SQL-92.

![ANSI/ISO SQL-92](..%2Fimages%2Fposts%2F2021-12-02-til-tsql-string-compare%2Fspec.jpeg)

## So why does it do this? 🙋‍♂️
The ANSI/ISO SQL-92, for those that don't know, requires padding for the strings used in comparisons so that their lengths match before comparing them. This padding affects the semantics of `WHERE` and `HAVING` clause predicates among other TSQL string comparisons. For example, TSQL would consider both `'A'` and `'A '` to be the same for most comparison operations.

The only exception to this is the `LIKE` predicate, because the purpose of this predicate (by definition) is to facilitate pattern searches rather than equality tests.

![TSQL](..%2Fimages%2Fposts%2F2021-12-02-til-tsql-string-compare%2Ftsql.jpeg)

## TL;DR ⏭
```sql
MBP:~ mtjb$ sqlcmd -b -S "127.0.0.1" -U sa -P 'password'
1>
1>
1> SELECT CASE WHEN 'A ' = 'A' THEN 1 ELSE 0 END
2> GO

-----------
          1

(1 rows affected)
1>
1>
1> SELECT CASE WHEN ' A' = 'A' THEN 1 ELSE 0 END
2> GO

-----------
          0

(1 rows affected)
1>
1>
1> SELECT CASE WHEN 'A ' LIKE 'A' THEN 1 ELSE 0 END
2> GO

-----------
          0

(1 rows affected)
1>
```