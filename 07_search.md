---
layout: page
title: 搜索
permalink: /search/
---
<input type="text" id="search-input" placeholder="搜索文章...">
<div id="search-results"></div>
<script src="/js/lunr.js"></script>
<script>
  let idx;
  fetch('/search.json')
    .then(res => res.json())
    .then(data => {
      idx = lunr(function() {
        this.ref('url');
        this.field('title');
        this.field('content');
        data.forEach(doc => this.add(doc));
      });
    });
  
  document.getElementById('search-input').addEventListener('keyup', function() {
    let results = idx.search(this.value);
    // 显示结果...
  });
</script>