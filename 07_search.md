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
  <div id="search-status"></div>
  <div id="search-results" class="search-results"></div>
</div>

<link rel="stylesheet" href="/assets/css/search.css">

<script src="https://cdn.jsdelivr.net/npm/lunr@2.3.9/lunr.min.js"></script>
<script>
  // 防抖函数
  function debounce(func, wait) {
    let timeout;
    return function executedFunction(...args) {
      const later = () => {
        clearTimeout(timeout);
        func(...args);
      };
      clearTimeout(timeout);
      timeout = setTimeout(later, wait);
    };
  }

  let idx = null;
  let postsData = [];

  // 1. 加载索引数据
  document.addEventListener('DOMContentLoaded', function() {
    fetch('/search.json')
      .then(response => {
        if (!response.ok) throw new Error('Network response was not ok');
        return response.json();
      })
      .then(data => {
        console.log('Loaded posts:', data.length); // 调试用
        postsData = data;
        
        // 2. 构建Lunr索引
        idx = lunr(function() {
          this.ref('id');
          this.field('title', { boost: 10 });
          this.field('content', { boost: 5 });
          this.field('date');
          
          data.forEach(function(doc) {
            this.add(doc);
          }, this);
        });
        
        document.getElementById('search-status').textContent = 
          `已索引 ${data.length} 篇文章，可以开始搜索`;
      })
      .catch(error => {
        console.error('Error loading search index:', error);
        document.getElementById('search-status').textContent = 
          '搜索索引加载失败，请刷新页面重试';
      });
  });

  // 3. 搜索功能
  const searchInput = document.getElementById('search-input');
  const searchResults = document.getElementById('search-results');
  
  function performSearch(query) {
    if (!idx) {
      searchResults.innerHTML = '<p>搜索索引尚未加载完成，请稍候...</p>';
      return;
    }
    
    if (query.length < 2) {
      searchResults.innerHTML = '<p>请输入至少2个字符进行搜索</p>';
      return;
    }
    
    try {
      const results = idx.search(query);
      
      if (results.length === 0) {
        searchResults.innerHTML = `
          <div class="no-results">
            <p>没有找到包含 "<strong>${query}</strong>" 的文章</p>
            <p>尝试其他关键词或减少搜索词</p>
          </div>
        `;
        return;
      }
      
      // 显示结果
      let resultsHtml = `
        <div class="results-header">
          <p>找到 ${results.length} 个结果：</p>
        </div>
      `;
      
      results.slice(0, 20).forEach(result => {
        const post = postsData.find(p => p.id === parseInt(result.ref));
        if (post) {
          // 高亮显示匹配内容（简化版）
          const excerpt = post.content.substring(0, 150) + '...';
          resultsHtml += `
            <article class="search-result">
              <h3><a href="${post.url}">${post.title}</a></h3>
              <p class="result-excerpt">${excerpt}</p>
              <p class="result-meta">发布于：${post.date}</p>
            </article>
          `;
        }
      });
      
      searchResults.innerHTML = resultsHtml;
      
    } catch (error) {
      console.error('Search error:', error);
      searchResults.innerHTML = '<p>搜索过程中出现错误，请重试</p>';
    }
  }

  // 添加防抖（300毫秒）
  searchInput.addEventListener('input', debounce(function(e) {
    const query = e.target.value.trim();
    document.getElementById('search-status').textContent = 
      query ? `正在搜索: "${query}"` : '输入关键词开始搜索';
    performSearch(query);
  }, 300));
</script>