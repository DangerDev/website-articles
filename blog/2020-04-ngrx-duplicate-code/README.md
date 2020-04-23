---
title: "NgRx: How to avoid duplicated code"
author: Johannes Hoppe
mail: johannes.hoppe@haushoppe-its.de
published: 2020-04-26
keywords:
  - NgRx
  - Redux
language: en
thumbnail: xxx.jpg
---

I have noticed more than once that it can easily happen that duplicated code occurs in an architecture with NgRx. 
In this post I want to highlight a few ideas that help me working with NgRx/Redux. 

## The starting point

I work under the assumption that redundant code is acceptable up to a certain limit. If we are over-engineer into the wrong direction and immediately find a generic solution, that generic solution may be worse than the original problem. So if you have two similar actions, then this is not a problem for me. Therefore I use the following example: Our application has to load data, and we need a loading indication and we want to display the last error in case of an error. Unfortunately we already have altered the existing code three times...