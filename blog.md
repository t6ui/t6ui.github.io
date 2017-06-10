---
layout: default
title: Blog
---

<div class="blog">
    <h1 class="page-heading">Z's Blog</h1>
	<ul class="posts">

	  {% for post in site.posts %}
	    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ post.url }}" title="{{ post.title }}">{{ post.title }}</a></li>
	  {% endfor %}
	</ul>
</div>