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
      
      // 关键：先注册中文支持，然后再使用
      if (typeof lunr !== 'undefined') {
        try {
          // 方法1：直接初始化
          const idx = lunr(function() {
            // 注册多语言支持
            if (typeof lunr.multi !== 'undefined') {
              lunr.multiLanguage('en', 'zh');
            }
            
            // 注册中文支持
            if (typeof lunr.zh !== 'undefined') {
              lunr.zh(this);
            }
            
            this.ref('id');
            this.field('text');
            
            this.add({ id: 1, text: '中文测试文章内容' });
            this.add({ id: 2, text: '英文测试文章内容' });
          });
          
          const results = idx.search('中文');
          output.innerHTML += `<p>✅ 中文搜索测试成功！找到 ${results.length} 个结果</p>`;
          console.log('搜索结果:', results);
          
        } catch (err) {
          output.innerHTML += `<p>❌ 错误: ${err.message}</p>`;
          console.error('完整错误:', err);
          
          // 尝试更简单的测试
          output.innerHTML += `<p>尝试简单测试...</p>`;
          try {
            const idx2 = lunr(function() {
              this.ref('id');
              this.field('text');
              this.add({ id: 1, text: '中文测试' });
            });
            const results2 = idx2.search('中文');
            output.innerHTML += `<p>基础搜索: 找到 ${results2.length} 个结果</p>`;
          } catch (err2) {
            output.innerHTML += `<p>基础搜索也失败: ${err2.message}</p>`;
          }
        }
      }
    }, 1000);
  </script>