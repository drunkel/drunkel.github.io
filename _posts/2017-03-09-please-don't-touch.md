---
layout: post
title:  "touch: true, dependent: :destroy, $!"
date:   2016-02-09 17:15:11 +0000
categories: rails
image:  /preview.jpg
---

A little while ago our team was faced with an issue: certain DELETE requests were taking extremely long to process, to the tune of over 1 minute! This post explores what can happen when you have a dependency with :touch and dependent: :destroy together in a :has_many relationship.

For this example I've extracted any domain specific implementation to a hypothetical application which manages a library:

* A library has many books.
* A book belongs to 1 library.
* When we update a book, we should update the `updated_at` timestamp for the library (touch).
* When we *destroy* a library, we should *destroy* all of the books in the library.


In this example we have a library with around 2 000 books. One day I decide to delete the library. What happens in the background?



