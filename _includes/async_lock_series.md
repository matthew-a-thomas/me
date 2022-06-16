This the {{ page.async_series_ordinal }} post in a short series about async
locks:

{% assign posts = site.posts | sort: "date" %}
{% for post in posts %}{% if post.async_series_ordinal %}
1. **{% if post == page %}{{ post.title }}{% else %}[{{ post.title }}]({{ post.url }}){% endif %}**{% if post == page %} (_this post_){% endif %}&mdash;{{ post.description }}{% endif %}{% endfor %}

### It turns out I haven't gotten this right yet! Stay tuned for the third post...

---