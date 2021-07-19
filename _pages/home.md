---
permalink: /
description: Matt's personal website
---
Welcome to my personal website. Here are my posts:

<ul>
{% for post in site.posts %}
<li>
  <p>
    <a href="{{ post.url }}">{{ post.title }}</a><br/>
    {{ post.description }}<br/>
    <i>{{ post.date | date: '%B %d, %Y' }}{% if post.category %} ({{ post.category }}){% endif %}</i>
  </p>
</li>
{% endfor %}
</ul>

[About me and this site]({% link _pages/about.md %}).