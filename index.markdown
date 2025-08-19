---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: home
author_profile: true
author: Brad
header:
  image: /assets/images/bdt_wide_thin.jpg
---
{%- assign latest_post = site.posts | first -%}
# {{ latest_post.title }}
{{ latest_post.content }}