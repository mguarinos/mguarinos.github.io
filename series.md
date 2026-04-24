---
layout: default
title: Series
permalink: /series/
---
<div class="container series-page">
  <header class="page-header">
    <h1>Series</h1>
    <p class="page-description">Multi-part guides that build up from scratch to a complete, working system.</p>
  </header>

  {%- if site.data.series.size == 0 -%}
    <p class="series-empty">No series yet.</p>
  {%- else -%}
  <div class="series-list">
    {%- for s in site.data.series -%}
      {%- assign series_posts = site.series | where: "series", s.name | sort: "series_part" -%}
      {%- assign first_post = series_posts | first -%}

      {%- assign total_words = 0 -%}
      {%- for post in series_posts -%}
        {%- assign pw = post.content | number_of_words -%}
        {%- assign total_words = total_words | plus: pw -%}
      {%- endfor -%}
      {%- assign total_rt = total_words | divided_by: 200 | at_least: 1 -%}

      <div class="series-card" id="{{ s.slug }}">

        {%- if s.image -%}
        <div class="series-card-image">
          <img src="{{ s.image | relative_url }}" alt="{{ s.name | escape }}">
        </div>
        {%- endif -%}

        <div class="series-card-header">
          <span class="series-card-label">Series</span>
          <h2 class="series-card-title">{{ s.name | escape }}</h2>
          {%- if s.description -%}
          <p class="series-card-desc">{{ s.description | escape }}</p>
          {%- endif -%}
          <div class="series-card-info">
            <span class="series-card-count">{{ series_posts.size }} parts</span>
            <span class="series-card-total-time">{{ total_rt }} min total</span>
            <span class="series-card-dates">{{ s.date }}</span>
            <span class="series-card-info-actions">
              {%- if s.github -%}
              <a href="{{ s.github }}" class="series-repo-btn" target="_blank" rel="noopener" aria-label="Source repository">
                <svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27.68 0 1.36.09 2 .27 1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.013 8.013 0 0016 8c0-4.42-3.58-8-8-8z"/></svg>
                Repo
              </a>
              {%- endif -%}
              <button class="series-share-btn" data-url="{{ site.url }}/series/#{{ s.slug }}" aria-label="Copy link to series">
                <svg width="13" height="13" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" aria-hidden="true"><path d="M10 13a5 5 0 0 0 7.54.54l3-3a5 5 0 0 0-7.07-7.07l-1.72 1.71"/><path d="M14 11a5 5 0 0 0-7.54-.54l-3 3a5 5 0 0 0 7.07 7.07l1.71-1.71"/></svg>
                <span class="series-share-label">Share</span>
              </button>
            </span>
          </div>
        </div>

        <div class="series-post-list-wrap{% if series_posts.size > 3 %} has-more{% endif %}">
          <ol class="series-post-list">
            {%- for post in series_posts -%}
              {%- assign post_words = post.content | number_of_words -%}
              {%- assign post_rt = post_words | divided_by: 200 | at_least: 1 -%}
            <li class="series-post-item">
              <a href="{{ post.url | relative_url }}" class="series-post-link">
                <span class="series-post-num">{{ post.series_part }}</span>
                <span class="series-post-body">
                  <span class="series-post-title">{{ post.title | escape }}</span>
                  {%- if post.description -%}
                  <span class="series-post-desc">{{ post.description | escape }}</span>
                  {%- endif -%}
                </span>
                <span class="series-post-time">{{ post_rt }} min</span>
              </a>
            </li>
            {%- endfor -%}
          </ol>
          {%- if series_posts.size > 3 -%}
          <div class="series-post-fade"></div>
          {%- endif -%}
        </div>

        <div class="series-card-bottom">
          {%- if series_posts.size > 3 -%}
          <button class="series-show-more" data-more="{{ series_posts.size | minus: 3 }}">Show {{ series_posts.size | minus: 3 }} more posts ↓</button>
          {%- else -%}
          <span></span>
          {%- endif -%}
          {%- if first_post -%}
          <a href="{{ first_post.url | relative_url }}" class="series-start-btn">Start reading →</a>
          {%- endif -%}
        </div>

      </div>
    {%- endfor -%}
  </div>
  {%- endif -%}
</div>

<script>
  document.querySelectorAll('.series-share-btn').forEach(function (btn) {
    btn.addEventListener('click', function () {
      var label = btn.querySelector('.series-share-label');
      navigator.clipboard.writeText(btn.dataset.url).then(function () {
        label.textContent = 'Copied!';
        btn.classList.add('series-share-btn--copied');
        setTimeout(function () {
          label.textContent = 'Share';
          btn.classList.remove('series-share-btn--copied');
        }, 2000);
      });
    });
  });

  document.querySelectorAll('.series-show-more').forEach(function (btn) {
    btn.addEventListener('click', function () {
      var wrap = btn.closest('.series-card').querySelector('.series-post-list-wrap');
      var isExpanded = wrap.classList.toggle('expanded');
      btn.textContent = isExpanded
        ? 'Show less ↑'
        : 'Show ' + btn.dataset.more + ' more posts ↓';
    });
  });
</script>
