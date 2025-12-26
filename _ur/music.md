---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: default
lang: UR
title: music
---
{% for tag in site.tags %}
  {% if tag[0] == "urdu technology" %}
  <ul>
    {% for post in tag[1] %}
      <li style="direction:rtl;font-size:24px;"><a href="{{ post.url | relative_url}}">{{ post.title }}</a></li>
    {% endfor %}
  </ul>
  {% endif %}
  {% endfor %}