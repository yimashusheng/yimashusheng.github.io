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


<script>
  // 确保基础lunr加载后，再加载中文支持
  window.addEventListener('load', function() {
    if (typeof lunr === 'undefined') {
      console.error('Lunr.js加载失败！');
      return;
    }
    
    console.log('Lunr基础库加载成功，版本:', lunr.version);
    
    // 动态加载中文支持
    const scripts = [
      'https://cdn.jsdelivr.net/npm/lunr-languages@1.10.0/lunr.stemmer.support.min.js',
      'https://cdn.jsdelivr.net/npm/lunr-languages@1.10.0/lunr.multi.min.js', 
      'https://cdn.jsdelivr.net/npm/lunr-languages@1.10.0/lunr.zh.min.js'
    ];
    
    let loadedCount = 0;
    
    scripts.forEach((src, index) => {
      const script = document.createElement('script');
      script.src = src;
      script.onload = function() {
        loadedCount++;
        console.log(`加载成功: ${src.split('/').pop()}`);
        
        if (loadedCount === scripts.length) {
          console.log('所有中文支持库已加载完成');
          console.log('lunr.zh:', typeof lunr.zh);
          
          // 所有脚本加载完成后，初始化搜索
          setTimeout(initSearch, 100);
        }
      };
      script.onerror = function() {
        console.error(`加载失败: ${src}`);
        // 尝试备用CDN
        const backupSrc = src.replace('unpkg.com', 'cdn.jsdelivr.net/npm');
        const backupScript = document.createElement('script');
        backupScript.src = backupSrc;
        backupScript.onload = script.onload;
        backupScript.onerror = function() {
          console.error(`备用CDN也失败: ${backupSrc}`);
          loadedCount++;
          if (loadedCount === scripts.length) {
            initSearch(); // 即使失败也继续初始化
          }
        };
        document.head.appendChild(backupScript);
      };
      document.head.appendChild(script);
    });
  });
</script>

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
  let chineseSupportLoaded = false;

  // 检查中文支持是否真正可用
  function checkChineseSupport() {
    chineseSupportLoaded = typeof lunr !== 'undefined' && 
                          typeof lunr.zh !== 'undefined' &&
                          typeof lunr.multi !== 'undefined' &&
                          typeof lunr.stemmer !== 'undefined';
    
    console.log('中文支持检查结果:', {
      lunr: typeof lunr,
      lunr_zh: typeof lunr.zh,
      lunr_multi: typeof lunr.multi,
      lunr_stemmer: typeof lunr.stemmer,
      allLoaded: chineseSupportLoaded
    });
    
    return chineseSupportLoaded;
  }

  // 初始化搜索
  function initSearch() {
    console.log('开始初始化搜索...');
    
    // 检查中文支持
    checkChineseSupport();
    
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
          // 构建索引
          idx = lunr(function() {
            // 如果中文支持可用，启用它
            if (chineseSupportLoaded) {
              console.log('启用中文分词支持...');
              try {
                // 先注册中文支持
                lunr.zh(lunr);
                // 然后使用
                this.use(lunr.zh);
                console.log('中文分词器已启用');
              } catch (err) {
                console.warn('启用中文分词器失败:', err);
                chineseSupportLoaded = false;
              }
            }
            
            this.ref('id');
            this.field('title', { boost: 15 });
            this.field('content', { boost: 5 });
            this.field('date', { boost: 1 });
            this.field('tags', { boost: 3 });
            
            // 为了提高中文搜索效果，移除英文停用词过滤器
            this.pipeline.remove(lunr.stopWordFilter);
            
            // 添加自定义中文停用词（可选）
            if (!chineseSupportLoaded) {
              console.log('使用基础搜索模式');
              // 添加简单的中文分词处理
              this.pipeline.before(lunr.stemmer, function(token) {
                // 简单的中文分词：将中文字符单独拆分
                const str = token.toString();
                if (/[\u4e00-\u9fa5]/.test(str)) {
                  // 将中文字符拆分成单个字符
                  return str.split('').map(function(char) {
                    return lunr.Token(char, token.metadata);
                  });
                }
                return token;
              });
            }
            
            // 添加文档
            data.forEach((doc, index) => {
              const docToAdd = {
                id: doc.id !== undefined ? doc.id : index,
                title: doc.title || '',
                content: doc.content || '',
                date: doc.date || '',
                tags: Array.isArray(doc.tags) ? doc.tags.join(' ') : (doc.tags || '')
              };
              this.add(docToAdd);
            });
            
            console.log('索引构建完成，文档数:', this.documentStore.length);
          });
          
          isInitialized = true;
          
          const statusMessage = chineseSupportLoaded 
            ? `✅ 已加载 ${data.length} 篇文章（中文搜索已启用）`
            : `⚠️ 已加载 ${data.length} 篇文章（使用基础搜索模式）`;
          
          document.getElementById('search-status').innerHTML = statusMessage;
          
          // 检查URL中的搜索参数
          const urlParams = new URLSearchParams(window.location.search);
          const queryParam = urlParams.get('q');
          if (queryParam && queryParam.trim().length > 1) {
            document.getElementById('search-input').value = queryParam;
            setTimeout(() => performSearch(queryParam), 300);
          }
          
        } catch (error) {
          console.error('构建索引时出错:', error);
          document.getElementById('search-status').textContent = 
            '搜索功能初始化失败: ' + error.message;
        }
      })
      .catch(error => {
        console.error('加载搜索索引时出错:', error);
        document.getElementById('search-status').textContent = 
          '无法加载搜索索引，请稍后重试';
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
            <li>${chineseSupportLoaded ? '✅ 中文搜索已启用' : '⚠️ 使用基础搜索模式'}</li>
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
    
    searchResults.innerHTML = '<p class="searching">🔍 搜索中...</p>';
    
    try {
      console.log('执行搜索:', query);
      
      // 对于中文搜索，需要特殊处理查询字符串
      let searchQuery = query;
      
      // 如果中文支持未加载，将中文字符拆分成单个字符进行搜索
      if (!chineseSupportLoaded && /[\u4e00-\u9fa5]/.test(query)) {
        // 将中文字符拆开并用OR连接，提高匹配率
        const chineseChars = query.split('').filter(char => /[\u4e00-\u9fa5]/.test(char));
        if (chineseChars.length > 0) {
          searchQuery = chineseChars.join(' ');
          console.log('转换后的查询:', searchQuery);
        }
      }
      
      const results = idx.search(searchQuery);
      console.log(`找到 ${results.length} 个结果`);
      
      if (results.length === 0) {
        searchResults.innerHTML = `
          <div class="no-results">
            <h3>🔍 没有找到相关文章</h3>
            <p>没有找到包含 "<strong>${query}</strong>" 的文章</p>
            <div class="suggestions">
              <p>建议：</p>
              <ul>
                <li>尝试使用其他关键词或同义词</li>
                <li>减少搜索词数量</li>
                <li>搜索单个词语而非完整句子</li>
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
          <p class="search-query">搜索: <strong>"${query}"</strong></p>
          ${!chineseSupportLoaded ? '<p class="search-mode">⚠️ 当前使用基础搜索模式，中文匹配可能不精确</p>' : ''}
        </div>
        <div class="results-list">
      `;
      
      results.slice(0, 20).forEach((result, index) => {
        const post = postsData.find(p => p.id === parseInt(result.ref));
        if (post) {
          // 高亮显示
          const regex = new RegExp(`(${query.split('').join('|')})`, 'gi');
          const highlightedTitle = post.title.replace(regex, '<mark>$1</mark>');
          
          // 生成摘要
          let excerpt = post.content || '';
          if (excerpt.length > 150) {
            // 尝试找到包含查询词的段落
            const queryIndex = excerpt.toLowerCase().indexOf(query.toLowerCase());
            if (queryIndex > -1) {
              const start = Math.max(0, queryIndex - 50);
              const end = Math.min(excerpt.length, queryIndex + 100);
              excerpt = '...' + excerpt.substring(start, end) + '...';
            } else {
              excerpt = excerpt.substring(0, 150) + '...';
            }
          }
          
          const highlightedExcerpt = excerpt.replace(regex, '<mark>$1</mark>');
          
          resultsHtml += `
            <article class="search-result">
              <div class="result-rank">#${index + 1}</div>
              <div class="result-content">
                <h4><a href="${post.url}">${highlightedTitle}</a></h4>
                <div class="result-excerpt">${highlightedExcerpt}</div>
                <div class="result-footer">
                  <span class="result-date">${post.date}</span>
                  ${post.tags && post.tags.length > 0 ? 
                    `<span class="result-tags">标签: ${Array.isArray(post.tags) ? post.tags.join(', ') : post.tags}</span>` : ''}
                </div>
              </div>
            </article>
          `;
        }
      });
      
      resultsHtml += '</div>';
      searchResults.innerHTML = resultsHtml;
      
    } catch (error) {
      console.error('搜索过程中出错:', error);
      searchResults.innerHTML = `
        <div class="error-message">
          <p>搜索过程中出现错误</p>
          <p><small>${error.message}</small></p>
        </div>
      `;
    }
  }

  // 页面交互
  document.addEventListener('DOMContentLoaded', function() {
    const searchInput = document.getElementById('search-input');
    
    // 实时搜索
    searchInput.addEventListener('input', debounce(function(e) {
      const query = e.target.value.trim();
      
      if (query) {
        document.getElementById('search-status').innerHTML = 
          `🔍 正在搜索: <strong>"${query}"</strong>`;
        
        const url = new URL(window.location);
        url.searchParams.set('q', query);
        window.history.pushState({}, '', url);
      } else {
        const modeText = chineseSupportLoaded ? '（中文搜索已启用）' : '（基础搜索模式）';
        document.getElementById('search-status').textContent = 
          '输入关键词开始搜索 ' + modeText;
        window.history.pushState({}, '', window.location.pathname);
      }
      
      performSearch(query);
    }, 400));
    
    // Enter键搜索
    searchInput.addEventListener('keypress', function(e) {
      if (e.key === 'Enter') {
        e.preventDefault();
        const query = e.target.value.trim();
        if (query) {
          document.getElementById('search-status').textContent = `搜索: "${query}"`;
          performSearch(query);
        }
      }
    });
    
    // 自动聚焦
    setTimeout(() => {
      searchInput.focus();
      const urlParams = new URLSearchParams(window.location.search);
      const queryParam = urlParams.get('q');
      if (queryParam) {
        searchInput.value = queryParam;
      }
    }, 500);
  });
</script>