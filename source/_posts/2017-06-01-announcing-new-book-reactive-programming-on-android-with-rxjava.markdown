---
layout: post
title: "Announcing New Book: Reactive Programming on Android with RxJava"
date: 2017-06-01 18:44:00 -0500
comments: true
categories: Android, RxJava
---

I'm very pleased to announce that a book that [Angus Huang](https://www.linkedin.com/in/ahuang13/) and I started writing a few months back, ["Reactive Programming on Android"](https://leanpub.com/reactiveandroid), is now published and available on LeanPub!

{% img center http://chrisarriola.me/images/rxjava_my_book_with_angus.jpg %}

## What is RxJava?

RxJava, the Java implementation of ReactiveX, was open sourced and introduced to the developer community by Netflix back in 2013. At Netflix, RxJava had arisen as a need to solve scaling issues created by their previous [one-size-fits-all API](https://medium.com/netflix-techblog/embracing-the-differences-inside-the-netflix-api-redesign-15fd8b3dc49d). 

The promise of reactive programming was that it would allow their teams to seamlessly compose complex asynchronous behavior into an easy-to-use API. Using these APIs, their client teams can then create custom end-points to optimize for the growing number of devices that Netflix supported. 

Indeed, RxJava has fulfilled its promise at Netflix and is now [the backbone of many of their back-end services](https://medium.com/netflix-techblog/reactive-programming-in-the-netflix-api-with-rxjava-7811c3a1496a). Additionally, RxJava has grown to be very popular in the Java community and is now the most starred Java repository on GitHub.

{% img center http://chrisarriola.me/images/github_rxjava_stars.png %}

Given RxJava's success at Netflix--the driver for [more than a third](http://time.com/3901378/netflix-internet-traffic/) of all internet traffic--it is no wonder that many companies are also adopting RxJava as part of their software stack. When it comes to Android app development, it has become the go-to library for enabling reactive programming.

## Why Write a Book?

We believe that reactive programing is shaping the way Android apps are being built. This is even evident in the direction Google is going with its new reactive-inspired [Android Architecture Components](https://developer.android.com/topic/libraries/architecture/index.html) announced recently at Google IO '17. 

On a personal note, I've been writing reactive apps since 2014 and have found it to increase my productivity and allowed me to write apps that are resilient and responsive. 

By writing this book, we are hoping to contribute in the Android community by educating fellow developers about what RxJava is and how they can benefit from it.

Interested in reading more? You can purchase or download a sample of the book [here](https://leanpub.com/reactiveandroid).