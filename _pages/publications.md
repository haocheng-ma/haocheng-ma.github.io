---
layout: page
permalink: /publications/
title: publications
nav: true
nav_order: 2
---

<p>
  Up-to-date publications are also available on
  <a href="https://scholar.google.com/citations?user=cdWqgwUAAAAJ&hl" target="_blank">Google Scholar</a>.
</p>

<!-- _pages/publications.md -->

<!-- Bibsearch Feature -->

{% include bib_search.liquid %}

<div class="publications">

<h2 class="pub-section thesis">PhD Thesis</h2>
{% bibliography --query @phdthesis --group_by none %}
<h2 class="pub-section thesis">Peer-reviewed Publications</h2>
{% bibliography %}

</div>
