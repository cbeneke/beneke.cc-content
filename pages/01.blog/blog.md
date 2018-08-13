---
title: Blog
header_image: grav_header.jpeg
child_type: item
---

{% set collection = page.collection() %}`
{% for child in collection %}
        {% include 'partials/blog_item.html.twig' with {'blog':page, 'page':child, 'truncate':true} %}
{% endfor %}