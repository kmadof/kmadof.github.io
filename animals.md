---
layout: page
title: "Links"
permalink: "/links/"
sidebar_link: true
---

<ul>
  {% for link in site.links %}
    <li>
      <a href="{{ link.url }}">{{ link.title }}</a>
      - {{ link.headline }}
    </li>
  {% endfor %}
</ul>