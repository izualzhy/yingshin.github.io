---
layout: article
title: Pearls
---

很喜欢《Programming Pearls》这本书，因此用了这个名字。介绍一些好看好玩的编程 paper、slides、video 等。

<div class="pearl-card-list grid--p-3">
{% for pearl in site.data.pearls.pearls %}
  <div class="cell">
    <div class="card pearl-card">
      <div class="card__content">
        <h3 class="card__header">
          <i class="fas fa-gem pearl-icon"></i>
          <a href="{{ pearl.url }}" target="_blank" rel="noopener noreferrer">
            {{ pearl.title }}
          </a>
        </h3>

        <div class="card__meta">
          <a class="tag-button" href="{{ '/tags.html' | relative_url }}#{{ pearl.tag | url_encode }}">
            <i class="fas fa-tag fa-xs"></i> {{ pearl.tag }}
          </a>
        </div>

        <div class="card__note">
          {{ pearl.note | markdownify }}
        </div>
      </div>
    </div>
  </div>
{% endfor %}
</div>
