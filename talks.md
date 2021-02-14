---
layout: page
title: Talks
permalink: /talks/
tags: talks
---

I've given a few talks over the past couple of years, mostly around information security.

{% assign ordered_talks = site.talks | sort:"order" %}
{% for talk in ordered_talks %}
<h3><a href="{{ project.url }}" class="header-link">{{ talk.name }}</a></h3>
<p>{{ talk.content }}</p>
{% endfor %}