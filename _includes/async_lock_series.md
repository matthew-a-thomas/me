This is the {{ page.async_series_ordinal }} post in a short series about async
locks:

{% assign posts = site.posts | sort: "title" | sort: "date" %}
{% for post in posts %}{% if post.async_series_ordinal %}
1. **{% if post == page %}{{ post.title }}{% else %}[{{ post.title }}]({{ post.url }}){% endif %}**{% if post == page %} (_this post_){% endif %}&mdash;{{ post.description }}{% endif %}{% endfor %}

---