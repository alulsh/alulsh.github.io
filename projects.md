---
layout: page
title: Projects
permalink: /projects/
tags: projects
---

{% assign ordered_projects = site.projects | sort:"order" %}
## Security

{% for project in ordered_projects %}
  {% if project.type == 'security' %}
<h3>{{ project.name }}</h3>
<p> {{ project.content }} </p>
  {% endif %}
{% endfor %}

## Maps

{% for project in ordered_projects %}
  {% if project.type == 'maps' %}
<h3>{{ project.name }}</h3>
<p> {{ project.content }} </p>
  {% endif %}
{% endfor %}

## Community

{% for project in ordered_projects %}
  {% if project.type == 'community' %}
<h3>{{ project.name }}</h3>
<p> {{ project.content }} </p>
  {% endif %}
{% endfor %}
