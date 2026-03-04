---
layout: page
title: Archive
---

<div class="archive">
  {% assign posts_by_year = site.posts | group_by_exp: "post", "post.date | date: '%Y'" %}
  {% for year_group in posts_by_year %}
    <h2>{{ year_group.name }}</h2>
    <ul>
      {% for post in year_group.items %}
        <li>
          <span class="post-date">{{ post.date | date: "%b %d" }}</span>
          &nbsp;—&nbsp;
          <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
          {% if post.tags.size > 0 %}
            {% for tag in post.tags %}
              <span class="tag" style="font-size:0.7rem;">{{ tag }}</span>
            {% endfor %}
          {% endif %}
        </li>
      {% endfor %}
    </ul>
  {% endfor %}
</div>
