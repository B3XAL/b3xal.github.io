---
layout: default
---

<div class="home">

  <h1 class="page-heading">Posts en Español</h1>

  <ul class="post-list">
    {% assign posts_es = site.collections['posts-es'].docs | sort: 'date' | reverse %}
    {% for post in posts_es %}
      <li>
        <span class="post-meta">{{ post.date | date: "%b %-d, %Y" }}</span>
        <h2>
          <a class="post-link" href="{{ post.url | prepend: site.baseurl }}">{{ post.data.title }}</a>
        </h2>
      </li>
    {% endfor %}
  </ul>

  <p class="rss-subscribe">subscribe <a href="{{ "/feed.xml" | prepend: site.baseurl }}">via RSS</a></p>

</div>
