---
layout: default
title: Posts in English
---

<div class="home">

  <h1 class="page-heading">Posts in English</h1>

  <ul class="post-list">
    {% assign posts_en = site.collections['posts-en'].docs | sort: 'date' | reverse %}
    {% for post in posts_en %}
      <li>
        <span class="post-meta">{{ post.date | date: "%b %-d, %Y" }}</span>
        <h2>
          <a href="{{ post.url }}">{{ post.data.title }}</a>
        </h2>
      </li>
    {% endfor %}
  </ul>

  <p class="rss-subscribe">Subscribe <a href="{{ "/feed.xml" }}">via RSS</a></p>

  <nav>
    <a href="/es/">Español</a> | <a href="/en/">English</a>
  </nav>

</div>
