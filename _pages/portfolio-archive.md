---
layout: archive
title: "Portfolio"
permalink: /portfolio/
author_profile: true
excerpt: "A selection of recent projects."
header:
  overlay_color: "#000"
  overlay_filter: "0.5"
  overlay_image: keyboard.jpg
  caption: "Photo credit: [**Unsplash**](https://unsplash.com)"
---

{% include base_path %}

<div class="grid__wrapper">
  {% for post in site.portfolio %}
    {% include archive-single.html type="grid" %}
  {% endfor %}
</div>