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
<script src="https://unpkg.com/lunr-languages@1.10.0/lunr.stemmer.support.min.js"></script>
<script src="https://unpkg.com/lunr-languages@1.10.0/lunr.multi.min.js"></script>
<script src="https://unpkg.com/lunr-languages@1.10.0/lunr.zh.min.js"></script>

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

  // 高亮显示函数
  function highlightText(text, query) {
    if (!query) return text;
    
    const regex = new RegExp(`(${query.replace(/[.*+?^${}()|[\]\\]/g, '\\$&')})`, 'gi');
    return text.replace(regex, '<mark>$1</mark>');
  }

  // 生成摘要
  function generateExcerpt(content, query, length = 200) {
    if (!query) {
      return content.substring(0, length) + '...';
    }
    
    const lowerContent = content.toLowerCase();
    const lowerQuery = query.toLowerCase();
    const queryIndex = lowerContent.indexOf(lowerQuery);
    
    if (queryIndex === -1) {
      return content.substring(0, length) + '...';
    }
    
    // 让关键词出现在摘要中间位置
    const start = Math.max(0, queryIndex - Math.floor(length / 2));
    const end = Math.min(content.length, start + length);
    
    let excerpt = content.substring(start, end);
    if (start > 0) excerpt = '...' + excerpt;
    if (end < content.length) excerpt = excerpt + '...';
    
    return excerpt;
  }

  // 主要变量
  let idx = null;
  let postsData = [];
  let isInitialized = false;

  // 1. 初始化搜索
  function initSearch() {
    console.log('正在加载搜索索引...');
    
    fetch('/search.json')
      .then(response => {
        if (!response.ok) {
          throw new Error(`HTTP error! status: ${response.status}`);
        }
        return response.json();
      })
      .then(data => {
        console.log('成功加载文章数:', data.length);
        
        if (data.length === 0) {
          document.getElementById('search-status').textContent = '没有找到可搜索的文章';
          return;
        }
        
        postsData = data;
        
        try {
          // 2. 使用中文支持构建Lunr索引
          idx = lunr(function() {
            // 启用中文分词
            this.use(lunr.zh);
            
            this.ref('id');
            this.field('title', { boost: 15 });    // 标题权重最高
            this.field('content', { boost: 5 });   // 内容权重次之
            this.field('date', { boost: 1 });      // 日期权重最低
            
            // 添加停用词（中文常见停用词）
            this.pipeline.remove(lunr.stopWordFilter);
            
            // 添加文档
            data.forEach((doc, index) => {
              // 确保每个文档都有id
              if (doc.id === undefined) {
                doc.id = index;
              }
              this.add(doc);
            });
          });
          
          console.log('Lunr索引构建完成');
          isInitialized = true;
          
          document.getElementById('search-status').textContent = 
            `已加载 ${data.length} 篇文章，可以开始搜索`;
          
          // 检查是否有URL参数
          const urlParams = new URLSearchParams(window.location.search);
          const queryParam = urlParams.get('q');
          if (queryParam) {
            document.getElementById('search-input').value = queryParam;
            performSearch(queryParam);
          }
          
        } catch (error) {
          console.error('构建Lunr索引时出错:', error);
          document.getElementById('search-status').textContent = 
            '搜索功能初始化失败，请刷新页面重试';
        }
      })
      .catch(error => {
        console.error('加载搜索索引时出错:', error);
        document.getElementById('search-status').textContent = 
          '无法加载搜索索引，请检查网络连接';
      });
  }

  // 3. 执行搜索
  function performSearch(query) {
    const searchResults = document.getElementById('search-results');
    
    if (!query || query.trim().length === 0) {
      searchResults.innerHTML = `
        <div class="search-prompt">
          <h3>💡 搜索提示</h3>
          <ul>
            <li>输入关键词搜索文章内容</li>
            <li>支持中文分词搜索</li>
            <li>支持多个关键词搜索（用空格分隔）</li>
            <li>搜索结果按相关性排序</li>
          </ul>
        </div>
      `;
      return;
    }
    
    query = query.trim();
    
    if (!isInitialized || !idx) {
      searchResults.innerHTML = '<p>搜索功能正在初始化，请稍候...</p>';
      return;
    }
    
    if (query.length < 2) {
      searchResults.innerHTML = '<p>请输入至少2个字符进行搜索</p>';
      return;
    }
    
    // 显示搜索中状态
    searchResults.innerHTML = '<p class="searching">正在搜索中...</p>';
    
    try {
      // 执行搜索
      const results = idx.search(query);
      console.log(`搜索 "${query}" 找到 ${results.length} 个结果`);
      
      if (results.length === 0) {
        searchResults.innerHTML = `
          <div class="no-results">
            <h3>🔍 没有找到相关文章</h3>
            <p>没有找到包含 "<strong>${query}</strong>" 的文章</p>
            <div class="suggestions">
              <p>建议：</p>
              <ul>
                <li>尝试使用其他关键词</li>
                <li>减少搜索词数量</li>
                <li>检查拼写是否正确</li>
                <li>使用更通用的词语</li>
              </ul>
            </div>
          </div>
        `;
        return;
      }
      
      // 显示搜索结果
      let resultsHtml = `
        <div class="results-header">
          <h3>📚 搜索结果（共 ${results.length} 条）</h3>
          <p class="search-query">搜索关键词: <strong>"${query}"</strong></p>
        </div>
        <div class="results-list">
      `;
      
      // 限制显示数量
      const maxResults = 20;
      const displayResults = results.slice(0, maxResults);
      
      displayResults.forEach((result, index) => {
        const post = postsData.find(p => p.id === parseInt(result.ref));
        if (post) {
          // 生成高亮的标题和摘要
          const highlightedTitle = highlightText(post.title, query);
          const excerpt = generateExcerpt(post.content, query);
          const highlightedExcerpt = highlightText(excerpt, query);
          
          resultsHtml += `
            <article class="search-result">
              <div class="result-rank">#${index + 1}</div>
              <div class="result-content">
                <h4><a href="${post.url}">${highlightedTitle}</a></h4>
                <div class="result-excerpt">${highlightedExcerpt}</div>
                <div class="result-footer">
                  <span class="result-url">${window.location.origin}${post.url}</span>
                  <span class="result-date">${post.date}</span>
                </div>
              </div>
            </article>
          `;
        }
      });
      
      if (results.length > maxResults) {
        resultsHtml += `
          <div class="more-results">
            <p>还有 ${results.length - maxResults} 个结果未显示，请尝试更精确的搜索词</p>
          </div>
        `;
      }
      
      resultsHtml += '</div>'; // 关闭results-list
      searchResults.innerHTML = resultsHtml;
      
    } catch (error) {
      console.error('搜索过程中出错:', error);
      searchResults.innerHTML = `
        <div class="error-message">
          <p>搜索过程中出现错误: ${error.message}</p>
          <p>请刷新页面重试或联系管理员</p>
        </div>
      `;
    }
  }

  // 4. 事件监听
  document.addEventListener('DOMContentLoaded', function() {
    // 初始化搜索
    initSearch();
    
    const searchInput = document.getElementById('search-input');
    
    // 实时搜索（带防抖）
    searchInput.addEventListener('input', debounce(function(e) {
      const query = e.target.value.trim();
      
      if (query) {
        document.getElementById('search-status').textContent = 
          `正在搜索: "${query}"`;
        
        // 更新URL但不刷新页面
        const url = new URL(window.location);
        url.searchParams.set('q', query);
        window.history.pushState({}, '', url);
      } else {
        document.getElementById('search-status').textContent = 
          '输入关键词开始搜索';
        window.history.pushState({}, '', window.location.pathname);
      }
      
      performSearch(query);
    }, 350)); // 350ms防抖
    
    // 支持Enter键搜索
    searchInput.addEventListener('keypress', function(e) {
      if (e.key === 'Enter') {
        const query = e.target.value.trim();
        if (query) {
          performSearch(query);
        }
      }
    });
    
    // 页面加载时自动聚焦到搜索框
    searchInput.focus();
  });
</script>