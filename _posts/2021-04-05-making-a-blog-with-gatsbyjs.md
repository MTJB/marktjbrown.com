---
layout: post
title:  Why (and how) I moved to GatsbyJS
description: Honestly - the main reason I chose to move away from Wordpress and to GatsbyJS wasn't to upskill in the...
date:   2021-04-05 15:01:35 +0300
image:  '/images/posts/2021-04-05-making-a-blog-with-gatsbyjs/moving.jpeg'
tags:   [blogging]
---

## ⁉️ Why?
Honestly - the main reason I chose to move away from Wordpress and to [GatsbyJS](https://www.gatsbyjs.com/) wasn't to upskill in the [#2 Web Framework from 2020](https://insights.stackoverflow.com/survey/2020#technology-web-frameworks) (according to [stackoverflow](https://stackoverflow.com/)) - but just because.. I liked it better.

## 🤔 What is Gatsby?
GatsbyJS is free, open-source, and React-based. It's desiged to help developers build performant websites using static site generation. With my skillset in frontend development (read: none), this perked my ears up - up until now I hadn't really liked the appearance of my blog (entirely through fault of my own), and the [Novella](https://github.com/narative/gatsby-theme-novela) theme made it all sound so straightforward to produce something I liked the look and feel of.

## ⚡️ What are static sites?
Although new to me, static sites are not a new concept - it's just some HTML, CSS, and Javascript. There is no rendering at runtime, and there is no server-side code, no database etc. Having first-hand experience at writing some SQL injection bugs, I knew that could only be a good thing for this blog.

A static site generator is a tool that generates static sites (🤯 - I know), during _build time_. Then, once loaded, React takes over, and you have a single-page application!

Some of the other benefits of static sites allowed me to make improvements in speed, SEO, and security - without having to do anything! Allowing me to focus on writing posts, instead of pushing pixels about, or attempting to navigate through the maze that was Wordpress.

## 🙋‍♂️ What if you don't know React or GraphQL - can you still use Gatsby?
It certainly helps to know more about these to improve and build upon the starter projects - but I still know almost nothing, i've had to tweak a few _.js_ files to add dependencies, and I couldn't yet even tell you where the GraphQL code is 😅 - most of my contributions have been in markdown, which I'm fairly comfortable with 🤷‍♂️

## 😎 So what are the _real_ reasons I moved to GatsbyJS?
I'm just a simple, lazy guy - switching to GatsbyJS looked like it would allow me to create a blog that I liked, importantly with _no_ effort required from myself to learn any more of _the codes_ (at least initially). But also, out of the box it gave me;

1. Dark-themed code snippets 💯
1. Dark theme toggle on my entire blog - because everyone knows [light theme is bad for you](https://i.redd.it/oa59qwy5sio21.png).
1. Ability to write posts in [markdown](https://github.com/MTJB/blog_marktjbrown/blob/master/content/posts/2021-04-05-moving-to-gatsby/index.md) 😌
1. A nicer-looking site, in my opinion 😍 (but you know what they say about opinions...).

```java
// Method/Variable names have been changed to prevent plagarism detection
private boolean isEven(int number) {
    String numString = String.valueOf(number);
    String lastChar = numString.substring(numString.length() - 1);

    if (lastChar == '0' || lastChar == '2' || lastChar == '4' || lastChar == '6' || lastChar == '8') {
        return true;
    }

    return false;
}
```

## 👨🏻‍🏫 How to create a new GatsbyJS blog
I use `npm` alongside the `gatsby-cli` to manage my dependencies and build my site for testing etc, therefore you will need to ensure you have both installed and running.

#### Getting started with Gatsby Starter Novella
The theme I use for my blog is built on top of the [Novella](https://github.com/narative/gatsby-theme-novela) theme, who helpfully provide a starter repo - this can be installed like so;

```bash
gatsby new my-site-name https://github.com/narative/gatsby-starter-novela
```

That's almost everything(!). Now, the final step is to run a local developement server to allow you to make changes to your site;

```bash
gatsby develop
```

This will start your site locally and it can be accessed in the browser by navigating to `http://localhost:8000/`.

From here, all that's left to do is add yourself as an [Author](https://github.com/narative/gatsby-theme-novela#step-4-adding-an-author), add a [post](https://github.com/narative/gatsby-theme-novela#step-5-adding-a-post), and update some of the [site metadata](https://github.com/narative/gatsby-theme-novela#step-6-configuring-sitemetadata) - which the contributors have documented at the links provided.

#### Hosting your site on Gatsby Cloud
I chose to host my new site on Gatsby's own hosting solution - Gatsby Cloud, mainly because of their pricing tiers ([free](https://www.gatsbyjs.com/pricing/) for small projects!) but also because they provide a Real-time Preview, Content Management System (CMS) integration, and automated [Lighthouse](https://developers.google.com/web/tools/lighthouse/) reporting on each deployed version of my site. On top of all this, for me to add new blog posts - I just push to my [GitHub](https://github.com/MTJB/blog_marktjbrown) repo which is pre-authenticated with Gatsby Cloud and the commit hooks deploy a new version of my site automatically!

![Gatsby]({{site.baseurl}}/images/posts/2021-04-05-making-a-blog-with-gatsbyjs/moving.jpeg)

## 💅 Conclusion
Moving to GatsbyJS, for me, was relatively pain free - I was up and running with a clone of the Novella theme in minutes, and I only had 5 posts within Wordpress to move over - so a simple copy and paste job worked for me! With this change, I now have a blog I prefer the look of and more importantly, one that I find easier and faster to contribute to. At times, I found Wordpress quite heavyweight to add new content to, whereas now I can add new posts to my blog just by making a few new files in my repository and pushing it to Git!