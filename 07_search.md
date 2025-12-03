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
<script src="/assets/js/lunr.min.js"></script>
<!-- 中文分词支持（必须按顺序引入） -->
<script src="/assets/js/lunr.stemmer.support.min.js"></script>
<script src="/assets/js/lunr.multi.min.js"></script>
<script src="/assets/js/lunr.zh.min.js"></script>

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
  let isInitialized = false;

  // 检查中文分词支持是否可用
  function isChineseSupportAvailable() {
    return typeof lunr !== 'undefined' && 
           typeof lunr.zh !== 'undefined' &&
           typeof lunr.multi !== 'undefined';
  }

  // 初始化搜索
  function initSearch() {
    console.log('正在加载搜索索引...');
    
    fetch('/search.json')
      .then(response => {
        if (!response.ok) throw new Error(`HTTP错误! 状态: ${response.status}`);
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
          // 检查中文支持
          const hasChineseSupport = isChineseSupportAvailable();
          console.log('中文分词支持:', hasChineseSupport ? '已启用' : '未启用，使用英文搜索');
          
          // 构建Lunr索引
          idx = lunr(function() {
            // 如果中文支持可用，使用中文分词
            if (hasChineseSupport) {
              this.use(lunr.zh);
            } else {
              console.warn('中文分词支持未加载，使用英文搜索模式');
            }
            
            this.ref('id');
            this.field('title', { boost: 15 });
            this.field('content', { boost: 5 });
            this.field('date', { boost: 1 });
            
            // 添加停用词
            this.pipeline.remove(lunr.stopWordFilter);
            
            // 添加自定义中文停用词（简单版）
            if (!hasChineseSupport) {
              const chineseStopWords = ['的', '了', '在', '是', '我', '有', '和', '就', 
                '不', '人', '都', '一', '一个', '上', '也', '很', '到', '说', '要', '去',
                '你', '会', '着', '没有', '看', '好', '自己', '这'];
              
              this.pipeline.add(function(token, tokenIndex, tokens) {
                const tokenStr = token.toString();
                if (chineseStopWords.includes(tokenStr)) {
                  return null;
                }
                return token;
              });
            }
            
            // 添加文档
            data.forEach((doc, index) => {
              if (doc.id === undefined) doc.id = index;
              this.add(doc);
            });
          });
          
          console.log('Lunr索引构建完成');
          isInitialized = true;
          
          const statusText = hasChineseSupport 
            ? `已加载 ${data.length} 篇文章（中文搜索已启用）`
            : `已加载 ${data.length} 篇文章（英文搜索模式）`;
          
          document.getElementById('search-status').textContent = statusText;
          
          // 检查URL参数
          const urlParams = new URLSearchParams(window.location.search);
          const queryParam = urlParams.get('q');
          if (queryParam) {
            document.getElementById('search-input').value = queryParam;
            setTimeout(() => performSearch(queryParam), 500);
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
          '无法加载搜索索引: ' + error.message;
      });
  }

  // 执行搜索
  function performSearch(query) {
    const searchResults = document.getElementById('search-results');
    
    if (!query || query.trim().length === 0) {
      searchResults.innerHTML = `
        <div class="search-prompt">
          <h3>💡 搜索提示</h3>
          <ul>
            <li>输入关键词搜索文章内容</li>
            <li>支持中文和英文搜索</li>
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
      
      const maxResults = 20;
      const displayResults = results.slice(0, maxResults);
      
      displayResults.forEach((result, index) => {
        const post = postsData.find(p => p.id === parseInt(result.ref));
        if (post) {
          // 高亮显示
          const highlightedTitle = post.title.replace(
            new RegExp(`(${query})`, 'gi'), 
            '<mark>$1</mark>'
          );
          
          const excerpt = post.content.length > 150 
            ? post.content.substring(0, 150) + '...' 
            : post.content;
            
          const highlightedExcerpt = excerpt.replace(
            new RegExp(`(${query})`, 'gi'), 
            '<mark>$1</mark>'
          );
          
          resultsHtml += `
            <article class="search-result">
              <div class="result-rank">#${index + 1}</div>
              <div class="result-content">
                <h4><a href="${post.url}">${highlightedTitle}</a></h4>
                <div class="result-excerpt">${highlightedExcerpt}</div>
                <div class="result-footer">
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
      
      resultsHtml += '</div>';
      searchResults.innerHTML = resultsHtml;
      
    } catch (error) {
      console.error('搜索过程中出错:', error);
      searchResults.innerHTML = `
        <div class="error-message">
          <p>搜索过程中出现错误</p>
          <p>${error.message}</p>
        </div>
      `;
    }
  }

  // 页面加载
  document.addEventListener('DOMContentLoaded', function() {
    // 延迟初始化，确保所有脚本加载完成
    setTimeout(initSearch, 1000);
    
    const searchInput = document.getElementById('search-input');
    
    // 实时搜索
    searchInput.addEventListener('input', debounce(function(e) {
      const query = e.target.value.trim();
      
      if (query) {
        document.getElementById('search-status').textContent = 
          `正在搜索: "${query}"`;
        
        const url = new URL(window.location);
        url.searchParams.set('q', query);
        window.history.pushState({}, '', url);
      } else {
        document.getElementById('search-status').textContent = 
          '输入关键词开始搜索';
        window.history.pushState({}, '', window.location.pathname);
      }
      
      performSearch(query);
    }, 350));
    
    // Enter键搜索
    searchInput.addEventListener('keypress', function(e) {
      if (e.key === 'Enter') {
        e.preventDefault();
        const query = e.target.value.trim();
        if (query) {
          performSearch(query);
        }
      }
    });
    
    // 自动聚焦
    searchInput.focus();
  });
</script>