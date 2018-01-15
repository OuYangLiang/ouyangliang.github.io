---
layout: page
permalink: /category
---
<div id="categories">
    {% for cat in site.categories %}
    <div class="panel panel-default">
        <div class="panel-heading">
            <h3 class="panel-title" id="{{cat.first}}">{{cat.first}}</h3>
        </div>

        <div class="panel-body">
            <ul class="list-group">
            {% for f in cat.last %}
                <li class="list-group-item"><span class="blog-post-meta-category-page">{{ f.date |date: "%Y-%m-%d"}}&nbsp;&gt;&gt;&nbsp;</span><a href="{{f.url}}">{{ f.title }}</a></li>
            {% endfor %}
            </ul>
        </div>
    </div>
    {% endfor %}
</div>
