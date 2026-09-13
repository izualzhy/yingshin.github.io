---
layout: article
title: Pearls
---

很喜欢《Programming Pearls》这本书，因此用了这个名字。介绍一些好看好玩的编程 paper、slides、video 等。

{% assign pearl_groups = "talk|paper|course" | split: "|" %}
{% assign group_names = "Talks|Papers|Courses" | split: "|" %}

<div class="pearls">
{% for group in pearl_groups %}
{% assign group_index = forloop.index0 %}
{% assign group_pearls = site.data.pearls.pearls | where: "type", group %}
{% if group_pearls.size > 0 %}
<div class="pearl-group">
<h2 class="pearl-group__header"><span class="pearl-group__dot" aria-hidden="true"></span>{{ group_names[group_index] }}</h2>
<ul class="pearl-list">
{% for pearl in group_pearls %}
<li class="pearl-item">
<div class="pearl-item__title">
<a href="{{ pearl.url }}" target="_blank" rel="noopener noreferrer">{{ pearl.title }}</a>
<span class="pearl-item__tag">{{ pearl.tag }}</span>
</div>
<div class="pearl-item__note">
{{ pearl.note | markdownify }}
</div>
</li>
{% endfor %}
</ul>
</div>
{% endif %}
{% endfor %}
</div>
