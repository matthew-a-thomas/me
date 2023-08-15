---
permalink: /
description: Matt's personal website
---
Welcome to my personal website. Here are my posts:

<ul>
{% for post in site.posts %}
<li>
  <p>
    <b><a href="{{ post.url }}">{{ post.title }}</a></b>
    {% for series in site.series %}
    {% for path in series.paths %}
    {% if post.path == path %}
    (<i>Series: <b>{{ series.content }}</b></i>)
    {% endif %}
    {% endfor %}
    {% endfor %}
    <br/>
    &mdash;{{ post.description }}<br/>
    <i>{{ post.date | date: '%B %d, %Y' }}{% if post.category %} ({{ post.category }}){% endif %}</i>
  </p>
</li>
{% endfor %}
</ul>

[About me and this site]({% link _pages/about.md %}).