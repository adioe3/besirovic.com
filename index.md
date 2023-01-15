---
layout: index.liquid
---

{% for post in collections.posts.pages %}
<span class="date">{% if post.is_draft %}draft{% else %}{{ post.published_date | date: "%d %b %Y" }}{% endif %}</span><span class="sep"> - </span>[{{ post.title }}]({{ post.permalink }})
{% endfor %}
