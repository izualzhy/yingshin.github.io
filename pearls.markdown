---
layout: article
title: Programming Pearls
---

很喜欢《Programming Pearls》这本书，因此用了这个名字。介绍一些好看好玩的编程paper、slides、video等。

{% for pearls in site.data.pearls %}
### [{{pearls.title}}]({{pearls.url}})
🔗: [{{ pearls.tag }}](/tags.html#{{ pearls.tag }})  
📝: {{ pearls.note }}
<hr style="width:100%"/>
{% endfor %}
