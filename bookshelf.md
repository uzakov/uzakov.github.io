---
layout: post
title: Bookshelf
---

You can find below my bookshelf. It's divided into two sections: the books that have changed my perspective, taught me, affected my lifestyle and other books I recommend, that might not had such a big effect but nevertheless still interesting.

I've written a summary of things that summarise the key themes in these book. They are useful to help me remember the main points of each book.

### Recommended

{% for post in site.bookshelfRecommended %}

<h3><a href="{{ post.url }}">{{ post.title }}</a></h3>

{% endfor %}