---
layout: page
title: Projects
permalink: /projects/
---
Everything built on top of the [foundation](/foundation/) core, in build
order.

<div class="project-index-grid">
{% assign all_projects = site.projects | sort: "index" %}
{% for p in all_projects %}
  {% include project-card.html project=p %}
{% endfor %}
</div>
