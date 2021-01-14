---
layout: single
title: How I've created this blog - Part 1
category: how to
tags: jekyll github multilanguage
---

## Why I have started this

While doing my planning for this next calendar year, I have realized that one thing I always wanted to do but never really started was to **start blogging about technology** and other related topics from my daily basis job.

Also, one of the requirements for this blog was to **share my thoughts and experiences in both English and Portuguese**, once I always wanted to contribute not only to global community but also to help grow the local one, where not everyone is proficient in the English language.

The main complicator in all of this is that **I'm not a developer**. This should be done with low or none effort of development and customizations that my bare knowledge on HTML and CSS would be enough to successfully complete it.

## The solution: Jekyll and Github Pages

During this journey I have considered using Wordpress or Medium to share my thoughts but I got curious how to get one of those github.io domains and I met both Jekyll and Github Pages.

[Github Pages](https://pages.github.com/) are basically a website hosted on a Github repository based on static content. You just need to commit your changes to your repo and the content is served directly from there through a custom github.io subdomain, which is entirely free and easy to use. As you can host static content you can use your preferred static content generator, being Jekyll the most popular and recommended by Github where you just need to push your changes to the repository and the static content is automatically built using a an automated pipeline as long as you use pre-approved gems only.

[Jekyll](https://jekyllrb.com/) is based on Ruby and leverage Markdown and Liquid to create websites in a easy manner and, most importantly, **it is blog aware**: which will save lots of time during this development.

About the multi-language requirement, after a quick search I came across [jekyll-multiple-languages-plugin](https://github.com/kurtsson/jekyll-multiple-languages-plugin) which is easy to use and will completely attend to my requirements, allowing me to quickly create a blog with these requirements. The only detail is that this plugin isn't approved by Github and build process would require a little more effort as we'll discuss later.

## Creating the blog

### Creating a Github repository

I won't stick with the details of this topic as Github documentation has [a very detailed walkthrough](https://docs.github.com/en/free-pro-team@latest/github/working-with-github-pages/creating-a-github-pages-site) on how to achieve this. The important point 

### Selecting a Jekyll theme







As my first personal project of this year I've created this blog with this intent, but one thing I had in mind since the beginning was to target not only the global community but generate localized content in pt-br