---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: default
lang: UR
title: politics
---
<div class="content-blocks_ur">
{% for tag in site.tags %}
  {% if tag[0] == "urdu politics" %}
  <ul>
    {% for post in tag[1] %}
      <li style="direction:rtl;font-size:24px;"><a href="{{ post.url | relative_url}}">{{ post.title }}</a></li>
    {% endfor %}
  </ul>
  {% endif %}
  {% endfor %}
  </div>