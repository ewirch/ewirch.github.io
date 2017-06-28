---
layout: home
permalink: /
image:
  feature: wood-texture-1600x800.jpg
---


<div class="tiles">
	{% for post in site.categories.articles limit:4 %}
	  {% include post-grid.html %}
	{% endfor %}
</div>

<p style="clear:both;"/>

[More Articles...]({{site.url}}/articles/)