---
title: Online Hosted Instructions
permalink: index.html
layout: home
---

# Exercises

Hyperlinks to each of the lab exercises and demos are listed below.


{% assign labs = site.pages | where_exp:"page", "page.url contains '/Instructions/Labs'" %}
{% for activity in labs  %}
[{{ activity.lab.title }}{% if activity.lab.type %} - {{ activity.lab.type }}{% endif %}]({{ site.github.url }}{{ activity.url }})
{% endfor %}

