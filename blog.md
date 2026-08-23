---
layout: page
title: Blog
permalink: /blog/
---
Every weekly update, across all tracks, newest first.

{% assign all_posts = site.posts | sort: "date" | reverse %}
{% if all_posts.size > 0 %}
<div class="post-list">
  {% for post in all_posts %}
    {% include post-list-item.html post=post %}
  {% endfor %}
</div>
{% else %}
<p class="empty-note">No posts yet — check back soon.</p>
{% endif %}
