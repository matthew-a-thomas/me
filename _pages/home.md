---
permalink: /
description: Matt's personal website
---
Welcome to my personal website. Here are my posts:

{% for post in site.posts %}
* {{ post.date | date: '%B %d, %Y' }}&mdash;[{{ post.title }}]({{ post.url }}){% if post.category %} ({{ post.category }}){% endif %}
{% endfor %}

[About me and this site]({% link _pages/about.md %}).