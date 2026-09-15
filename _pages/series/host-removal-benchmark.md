---
layout: page
title: "Human Contamination Removal in Host-Associated Metagenomics"
permalink: /series/host-removal-benchmark/
---

{% assign posts = site.posts | where: "series", "host-removal-benchmark" | sort: "order" %}

<p>
This page lists all posts in the 5-day series on benchmarking human contamination removal
for host-associated metagenomic data. Using vaginal metagenomes as a case study, the series
covers host-removal strategies, reference databases, computational performance, and
microbial-read specificity.
</p>

<ul>
{% for post in posts %}
  <li>
    <strong>Day {{ post.order }}:</strong>
    <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
    <span style="opacity:0.75;">({{ post.date | date: "%b %d, %Y" }})</span>
  </li>
{% endfor %}
</ul>