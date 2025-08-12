---
layout: default
title: Posts en Español
---

<div class="home">

  <h1 class="page-heading">Posts en Español</h1>

  <ul class="post-list">
    {% assign posts_es = site.collections['posts-es'].docs | sort: 'date' | reverse %}
    {% for post in posts_es %}
      <li>
        <span class="post-meta">{{ post.date | date: "%b %-d, %Y" }}</span>
        <h2>
          <a href="{{ post.url }}">{{ post.data.title }}</a>
        </h2>
      </li>
    {% endfor %}
  </ul>

  <p class="rss-subscribe">Suscríbete <a href="{{ "/feed.xml" }}">vía RSS</a></p>

  <nav>
    <a href="/es/">Español</a> | <a href="/en/">English</a>
  </nav>

</div>
