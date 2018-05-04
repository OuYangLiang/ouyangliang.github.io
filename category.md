---
layout: default
permalink: /category
---
<div id="categories">
    {% for cat in site.categories %}
    <div class="panel panel-info hidden-category" id="{{cat.first}}_box" style="display: none">
        <div class="panel-heading">
            <h3 class="panel-title" id="{{cat.first}}">{{cat.first}}</h3>
        </div>

        <ul class="list-group">
          {% for f in cat.last %}
              <li class="list-group-item"><span class="blog-post-meta-category-page">{{ f.date |date: "%Y-%m-%d"}}&nbsp;&gt;&gt;&nbsp;</span><a href="{{f.url}}">{{ f.title }}</a></li>
          {% endfor %}
        </ul>
    </div>
    {% endfor %}
</div>
