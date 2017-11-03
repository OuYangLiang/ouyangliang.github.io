---
layout: page
permalink: /category
---
<div id="categories">
    {% for cat in site.categories %}

    <h3 id="{{cat[0]}}" class="bg-info">{{cat | first}}</h3>
    {% for f in cat.last %}
    <span class="blog-post-meta-category-page">{{ f.date |date: "%d %b, %Y"}}&gt;&gt;</span><a href="{{f.url}}">{{ f.title }}</a><br/>
    {% endfor %}
    {% endfor %}
</div>
