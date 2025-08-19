---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: home
author_profile: true
author: Brad
header:
  overlay_image: /assets/images/bdt_wide.jpeg
caption: ""
---
{%- assign latest_post = site.posts | first -%}
# {{ latest_post.title }}
{{ latest_post.content }}