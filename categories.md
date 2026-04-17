---
layout: default
title: Categories
permalink: /categories/
---
<div class="container categories-page">
  <header class="page-header">
    <h1>Categories</h1>
  </header>

  <div class="category-grid">
    {%- assign sorted_cats = site.categories | sort -%}
    {%- for cat_pair in sorted_cats -%}
      {%- assign cat_name = cat_pair[0] -%}
      {%- assign cat_posts = cat_pair[1] -%}
      <div class="category-card" id="{{ cat_name | slugify }}">
        <div class="category-card-header">
          <a href="#{{ cat_name | slugify }}" class="category-card-name">{{ cat_name }}</a>
          <span class="category-count">{{ cat_posts | size }}</span>
        </div>
        <ul class="category-post-list">
          {%- for post in cat_posts -%}
            <li>
              <a href="{{ post.url | relative_url }}">
                <time datetime="{{ post.date | date_to_xmlschema }}">{{ post.date | date: "%b %-d, %Y" }}</time>
                {{ post.title | escape }}
              </a>
            </li>
          {%- endfor -%}
        </ul>
      </div>
    {%- endfor -%}
  </div>
</div>
