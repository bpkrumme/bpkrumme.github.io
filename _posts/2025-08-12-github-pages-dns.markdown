---
layout: post
title:  "Configuring Custom DNS for GitHub Pages"
date:   2025-08-12
---
You should now see my site at www.redhatbrad.com instead of bpkrumme.github.io.

Let's cover how I did this using Route53 and GitHub:

The first thing we need is a valid domain which we can manage via Route53.  I used AWS to both purchase the domain and manage it, so it was done in one step.  Let's take a look at the Route53 dashboard.

![Screenshot of the Route53 Dashboard](/assets/images/route53_dashboard.png)

--Brad