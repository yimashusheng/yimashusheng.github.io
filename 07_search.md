---
layout: page
title: 搜索
permalink: /search/
---
<div class="search-container">
  <input 
    type="text" 
    id="search-input" 
    placeholder="输入关键词搜索文章..."
    aria-label="搜索文章"
  />
  <div id="search-status">正在加载搜索索引...</div>
  <div id="search-results" class="search-results"></div>
</div>

<link rel="stylesheet" href="/assets/css/search.css">

<!-- Lunr核心库 -->
<script src="https://cdn.jsdelivr.net/npm/lunr@2.3.9/lunr.min.js"></script>
<!-- 中文分词支持（必须按顺序引入） -->

<script src="https://cdn.jsdelivr.net/npm/lunr-languages@1.10.0/lunr.stemmer.support.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/lunr-languages@1.10.0/lunr.multi.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/lunr-languages@1.10.0/lunr.zh.min.js"></script>

<div id="output"></div>

<script>
    setTimeout(() => {
      const output = document.getElementById('output');
      
      output.innerHTML += `<p>lunr: ${typeof lunr}</p>`;
      output.innerHTML += `<p>lunr.zh: ${typeof lunr.zh}</p>`;
      output.innerHTML += `<p>lunr.multi: ${typeof lunr.multi}</p>`;
      
      if (typeof lunr !== 'undefined' && typeof lunr.zh !== 'undefined') {
        // 测试中文分词
        try {
          const idx = lunr(function() {
            lunr.zh(this);  // 关键：注册中文支持
            this.use(lunr.zh);
            this.ref('id');
            this.field('text');
            
            this.add({ id: 1, text: '中文测试文章内容' });
            this.add({ id: 2, text: '英文测试文章内容' });
          });
          
          const results = idx.search('中文');
          output.innerHTML += `<p>✅ 中文搜索测试成功！找到 ${results.length} 个结果</p>`;
        } catch (err) {
          output.innerHTML += `<p>❌ 中文搜索测试失败: ${err.message}</p>`;
        }
      } else {
        output.innerHTML += `<p>❌ 中文支持库未正确加载</p>`;
      }
    }, 2000);
  </script>