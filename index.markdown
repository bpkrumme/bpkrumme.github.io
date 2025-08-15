---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: home
author_profile: true
author:
  name: "Brad Krumme"
  avatar: "/assets/images/bk_profile.jpg"
  bio: "Solution Architect at Red Hat specializing in Ansible and the Ansible Automation Platform."
  location: USA
  links:
    - label: "LinkedIn"
      url: "https://www.linkedin.com/in/bradkrumme/"
    - label: "GitHub"
      url: "https://github.com/bpkrumme/"
---
{%- assign latest_post = site.posts | first -%}
  ## {{ latest_post.title }}
  {{ latest_post.content }}