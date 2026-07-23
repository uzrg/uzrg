---
layout: page
icon: fas fa-images
order: 5
title: Gallery
permalink: /gallery/
---

A running set of console screenshots from the lab — proof, not polish.
Organized by platform, then by system.

{% assign platforms = "Hyper-V Lab|VMware" | split: "|" %}

{% for platform in platforms %}
## {{ platform }}

{% assign items = site.data.gallery | where: "platform", platform %}
{% if items.size == 0 %}
<p class="text-muted fst-italic">No screenshots yet — this environment is still on the roadmap.</p>
{% else %}
{% assign categories = items | map: "category" | uniq %}
{% for cat in categories %}
### {{ cat }}

<div class="gallery-grid">
{% assign cat_items = items | where: "category", cat %}
{% for item in cat_items %}
  <figure class="gallery-item">
    <img src="{{ '/assets/img/gallery/' | append: item.file | relative_url }}" alt="{{ item.caption | escape }}">
    <figcaption>
      {% if item.post %}<a href="{{ item.post | relative_url }}">{{ item.caption }}</a>{% else %}{{ item.caption }}{% endif %}
    </figcaption>
  </figure>
{% endfor %}
</div>
{% endfor %}
{% endif %}
{% endfor %}
