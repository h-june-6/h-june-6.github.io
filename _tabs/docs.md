---
layout: page
title: 블로그/기술문서/리서치
icon: fas fa-pen-nib
order: 4
group: category
subgroup: docs-sub
---
{%- assign subs = site.tabs | where: 'group', 'docs-sub' -%}
{%- for st in subs -%}
<h2 class="h4 mt-4"><a href="{{ st.url | relative_url }}">{{ st.title }}</a></h2>
{%- assign posts = site.categories[st.category_key] -%}
{%- if posts and posts.size > 0 -%}
<ul class="content ps-0">
  {%- for post in posts -%}
  <li class="d-flex justify-content-between px-md-3">
    <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
    <span class="dash flex-grow-1"></span>
    <time class="text-muted small text-nowrap">{{ post.date | date: '%Y.%m.%d' }}</time>
  </li>
  {%- endfor -%}
</ul>
{%- else -%}
<p class="text-muted">아직 작성된 글이 없어요.</p>
{%- endif -%}
{%- endfor -%}
