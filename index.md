---
layout: default
---


<ul class="post-list">
  {% for post in site.posts %}
    <li>
    	<h2>
			<span class="post-meta">{{ post.date | date_to_long_string }}</span>
      		<a class="post-link" href="{{ post.url }}">{{ post.title }}</a>
      		<span class="post-meta">标签：{{ post.tags }}</span>
      	</h2>
    </li>
  {% endfor %}
</ul>