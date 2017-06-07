---
layout: post
title: "Announcing New Book: Reactive Programming on Android with RxJava"
date: 2017-06-12 18:44:00 -0500
comments: true
categories: Android, RxJava
---

I'm very pleased to announce that a book that [Angus Huang](https://www.linkedin.com/in/ahuang13/) and I started writing a few months back, ["Reactive Programming on Android"](https://leanpub.com/reactiveandroid), is now published and available on LeanPub!

{% img center http://chrisarriola.me/images/rxjava_my_book_with_angus.jpg %}

## What is RxJava?

RxJava—the Java implementation of ReactiveX—was open sourced and introduced to the developer community by Netflix back in 2013. At Netflix, RxJava had arisen as a need to solve scaling issues created by their previous [one-size-fits-all API](https://medium.com/netflix-techblog/embracing-the-differences-inside-the-netflix-api-redesign-15fd8b3dc49d). 

The promise of reactive programming was that it would allow their teams to seamlessly compose complex asynchronous behavior into an easy-to-use API. Using these APIs, their client teams can then create custom end-points to optimize for the growing number of devices that Netflix supported without having to deal with the intricacies of server-side concurrent programming. 

In a word, RxJava was supposed to **simplify writing concurrent code**.

Turns out, RxJava did fulfill its promise and is now [the backbone of many Netflix back-end services](https://medium.com/netflix-techblog/reactive-programming-in-the-netflix-api-with-rxjava-7811c3a1496a). 

Outside of Netflix, RxJava has been adopted in the Android community as developing mobile apps also has many concurrency needs. As of today, RxJava is the go-to library for enabling reactive programming on Android. Certainly, the number of stars it has on Github should be a strong signal.

{% img center http://chrisarriola.me/images/github_rxjava_stars.png RxJava is the most starred Java repository on Github as of June 2017 %}

## Why Write this Book?

We believe that reactive programing is shaping the way Android apps are being built. This is even evident in the direction Google is going with its new reactive-inspired [Android Architecture Components](https://developer.android.com/topic/libraries/architecture/index.html) announced recently at Google IO '17. With that said, we think it's important for Android developers to familiarize themselves with the reactive programming model.

This book is a collection of our knowledge on the subject taken from different sources around the web (i.e. blog posts, books, wikis, etc.). Our hope is that this book services as a solid foundation for Android developers who are new to RxJava and want to start integrating it into their apps.

If you are interested in learning more, you can purchase or download a sample of the book [here](https://leanpub.com/reactiveandroid). 

Got any questions? Leave a comment below!