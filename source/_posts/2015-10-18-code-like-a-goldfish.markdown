---
layout: post
title: "Code Like a Goldfish"
date: 2015-10-18 16:46:47 -0700
comments: true
categories: Software Design, Design Patterns, Talks
---

There is an underlying hidden requirement in creating software products. It doesn't just have to "work"; it needs to be molded in such a way that it can be changed, maintained, and scaled easily.

Last month, I gave a talk addressing this topic to the 1st cohort at [Telegraph Academy](http://www.telegraphacademy.com/)—an  immersive program that teaches underrepresented groups in tech on how to code. They were nearing their last week of the program so I thought it would be relevant to share some industry lessons.

**TL;DR**

1. Design early

	The biggest mistake beginners tend to make is not taking the time to think through the pros/cons of a software design. Designing early makes it easier to find flaws, it's much harder fixing those flaws once it's expressed in code.

2. Don't over optimize

	This doesn't just apply to algorithms (e.g. trying to get O(n * log n) performance vs O(n<sup>2</sup>) on a small dataset), but it also applies to software design. Don't over-engineer or over-genericize a problem if it doesn't have to be.

3. Don't repeat yourself (DRY)

	Code repetition means if something needs to be changed/fixed, it needs to be changed in all parts of the code which is error-prone. Have one source of truth and abstract where appropriate.

4. Think small

	Smaller classes, methods, and files are easier to maintain.

Check out the slides [here](https://speakerdeck.com/arriolac/code-like-a-goldfish).