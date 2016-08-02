---
layout: page
permalink: /category.html
---
<div id="categories">
    {% for cat in site.categories %}
    <h3 id="{{cat[0]}}"><span class="label label-primary">{{cat[0]}}</span></h3>
    {% for f in cat[1] %}
    <span class="blog-post-meta-category-page">{{ f.date |date: "%d %b, %Y"}}&gt;&gt;</span><a href="{{f.url}}">{{ f.title }}</a><br/>
    {% endfor %}
    {% endfor %}
</div>
