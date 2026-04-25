---
layout: article
title: Programming Pearls
---

很喜欢《Programming Pearls》这本书，因此用了这个名字。这里记录一些我看到的优质编程资源：slides、技术文章、paper、视频等，以及我觉得值得推荐的理由。

<div class="pearls-container">
{% for pearl in site.data.pearls %}
<article class="pearl-card">
  <header class="pearl-header">
    <h3 class="pearl-title">
      <a href="{{ pearl.url }}" target="_blank" rel="noopener noreferrer">
        {{ pearl.title }}
      </a>
    </h3>
    <span class="pearl-tag">{{ pearl.tag }}</span>
  </header>
  
  <div class="pearl-content">
    <p class="pearl-note">{{ pearl.note }}</p>
  </div>
  
  <footer class="pearl-footer">
    <a href="{{ pearl.url }}" class="pearl-link" target="_blank" rel="noopener noreferrer">
      阅读原文 →
    </a>
  </footer>
</article>
{% endfor %}
</div>

<style>
.pearls-container {
  display: grid;
  gap: 1.5rem;
  margin-top: 2rem;
}

.pearl-card {
  background: var(--background-color, #fff);
  border-left: 4px solid #fd7e14;
  border-radius: 6px;
  padding: 1.5rem;
  margin-bottom: 1rem;
  box-shadow: 0 2px 6px rgba(0, 0, 0, 0.1);
}

.pearl-card:hover {
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
}

.pearl-header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  margin-bottom: 1rem;
  gap: 1rem;
}

.pearl-title {
  margin: 0;
  font-size: 1.25rem;
  line-height: 1.4;
  flex: 1;
}

.pearl-title a {
  color: var(--text-color, #24292e);
  text-decoration: none;
}

.pearl-title a:hover {
  color: var(--primary-color, #0366d6);
}

.pearl-tag {
  display: inline-block;
  padding: 0.2rem 0.6rem;
  background: var(--accent-color, #f1f8ff);
  color: var(--primary-color, #0366d6);
  border-radius: 4px;
  font-size: 0.8rem;
  font-weight: 500;
  white-space: nowrap;
}

.pearl-content {
  margin-bottom: 1rem;
}

.pearl-note {
  margin: 0;
  line-height: 1.8;
  color: var(--text-color-light, #586069);
}

.pearl-footer {
  border-top: 1px solid var(--border-color, #eaecef);
  padding-top: 1rem;
}

.pearl-link {
  display: inline-flex;
  align-items: center;
  gap: 0.3rem;
  color: var(--primary-color, #0366d6);
  text-decoration: none;
  font-weight: 500;
  font-size: 0.9rem;
  transition: all 0.2s ease;
}

.pearl-link:hover {
  gap: 0.5rem;
  color: #0256c7;
}

@media (max-width: 768px) {
  .pearl-header {
    flex-direction: column;
  }
  
  .pearl-tag {
    align-self: flex-start;
  }
}
</style>
