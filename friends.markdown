---
layout: article
title: Friends
---

<ul class="friends-list">
{% for friend in site.data.friends %}
  <li class="friend">
    <span class="friend__num">{{ forloop.index | prepend: '0' | slice: -2, 2 }}</span>
    <span class="friend__main">
      <a class="friend__name" href="{{ friend.site_url }}" target="_blank" rel="noopener">{{ friend.name }}</a>
      {% if friend.desc %}<span class="friend__desc">{{ friend.desc }}</span>{% endif %}
    </span>
  </li>
{% endfor %}
</ul>
