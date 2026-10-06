---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: default
title: PlusPlus Cyber
---

# Home

PlusPlus Cyber is a cybersecurity and IT research initiative dedicated to strengthening small-scale infrastructure and digital operations.

This GitHub functions as a public lab notebook. We document network configurations, system hardening methods, VPN architectures, automation scripts, and security experiments to promote transparency and practical defense against evolving cyber threats.

We believe security improves when it is built in the open and stress-tested relentlessly.

## Lab Notes

<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
      <span> — {{ post.date | date: "%b %-d, %Y" }}</span>
    </li>
  {% endfor %}
</ul>
