---
layout: page
title: Research Tracks
permalink: /tracks/
---
Each track below is a self-contained line of work built on top of the
[foundation](/foundation/) RISC-V core, logged as weekly posts.

<div class="track-grid">
{% assign ordered_tracks = site.tracks_order %}
{% for slug in ordered_tracks %}
  {% assign t = site.tracks | where: "slug", slug | first %}
  {% if t %}
  <a class="track-card" href="{{ t.url | relative_url }}" style="--track-color: {{ t.color | default: 'var(--accent2)' }};">
    <span class="track-icon-badge">{% include track-icon.html slug=t.slug %}</span>
    <h3>{{ t.title }}</h3>
    <p>{{ t.summary }}</p>
  </a>
  {% endif %}
{% endfor %}
</div>
