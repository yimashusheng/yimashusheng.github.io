---
layout: page
title: 搜索
permalink: /search/
---
<div class="search-container">
  <div class="search-header">
    <h1>🔍 搜索文章</h1>
    <div class="search-box">
      <input 
        type="text" 
        id="search-input" 
        placeholder="输入关键词搜索（支持中文）..."
        autocomplete="off"
        aria-label="搜索文章"
      />
      <div class="search-actions">
        <button id="clear-search" aria-label="清空搜索">✕</button>
      </div>
    </div>
    <div id="search-stats" class="search-stats"></div>
  </div>
  
  <div id="search-results" class="search-results">
    <div class="initial-message">
      <h3>💡搜索提示</h3>
      <ul>
        <li>支持中文、英文、拼音搜索</li>
        <li>支持多个关键词（空格分隔）</li>
        <li>支持模糊匹配和拼写容错</li>
        <li>搜索结果按相关性排序</li>
      </ul>
    </div>
  </div>
</div>

<!-- 引入Minisearch -->
<script src="https://cdn.jsdelivr.net/npm/minisearch@6.0.0/dist/umd/index.min.js"></script>

<script>


  // 配置参数
  const SEARCH_CONFIG = {
    debounceTime: 400,          // 防抖时间(毫秒)
    minQueryLength: 2,          // 最小搜索字符数
    maxResults: 50,             // 最大显示结果数
    fuzzy: 0.2,                 // 模糊匹配强度(0-1)
    boost: {                    // 字段权重
      title: 3,
      content: 1,
      tags: 2
    }
  };

  // 全局变量
  let searchIndex = null;
  let postsData = [];
  let isIndexReady = false;

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

  // 初始化搜索
  async function initSearch() {
    try {
      console.log('正在加载搜索索引...');
      updateStats('正在加载搜索索引...');
      
      // 1. 加载文章数据
      const response = await fetch('/search-index.json');
      if (!response.ok) {
        throw new Error(`HTTP错误 ${response.status}`);
      }
      
      const data = await response.json();
      postsData = data.posts || [];
      
      if (postsData.length === 0) {
        updateStats('没有找到可搜索的文章');
        return;
      }
      
      console.log(`成功加载 ${postsData.length} 篇文章`);
      
      // 2. 创建搜索索引
	// 替换原来的 searchIndex 创建部分
	searchIndex = new MiniSearch({
	  fields: ['title', 'content', 'tags'],
	  storeFields: ['id', 'title', 'url', 'date', 'excerpt', 'tags'],
	  searchOptions: {
	    boost: SEARCH_CONFIG.boost,
	    fuzzy: SEARCH_CONFIG.fuzzy,
	    prefix: true,
	    combineWith: 'OR'  // 改为 OR，提高召回率
	  },
	  // 改进的中文分词配置
	  tokenize: (text) => {
	    if (!text) return [];
	    
	    const str = text.toString();
	    const tokens = [];
	    
	    // 处理英文单词
	    const englishTokens = str.match(/[a-zA-Z0-9\u4e00-\u9fa5]+/g) || [];
	    
	    // 对于较长的中文字符串，进行简单的中文分词（按字符分割）
	    englishTokens.forEach(token => {
	      if (token.match(/[\u4e00-\u9fa5]/)) {
	        // 如果是中文字符串，可以按字符分割或者使用二元分词
	        if (token.length > 2) {
	          // 二元分词：对于长中文，创建相邻字符对
	          for (let i = 0; i < token.length - 1; i++) {
	            tokens.push(token.substring(i, i + 2));
	          }
	        }
	        // 同时保留完整的中文词汇
	        tokens.push(token);
	      } else {
	        tokens.push(token.toLowerCase());
	      }
	    });
	    
	    return tokens;
	  },
	  processTerm: (term) => {
	    if (!term) return '';
	    
	    // 保留所有中文字符和英文数字
	    return term.replace(/[^\w\u4e00-\u9fa5]/g, '').toLowerCase();
	  }
	});
      
      // 3. 添加文档到索引
      console.log('正在构建搜索索引...');
      
      // 批量添加，避免阻塞UI
      const batchSize = 100;
      for (let i = 0; i < postsData.length; i += batchSize) {
        const batch = postsData.slice(i, i + batchSize);
        searchIndex.addAll(batch);
        
        // 更新进度
        const progress = Math.min(100, Math.round((i + batch.length) / postsData.length * 100));
        updateStats(`正在构建索引... ${progress}%`);
        
        // 让出控制权，避免阻塞
        if (i + batchSize < postsData.length) {
          await new Promise(resolve => setTimeout(resolve, 0));
        }
      }
      
      isIndexReady = true;
      
      // 4. 显示就绪状态
      updateStats(`✅ 已索引 ${postsData.length} 篇文章，支持中文搜索`);
      
      // 5. 检查URL中的搜索参数
      const urlParams = new URLSearchParams(window.location.search);
      const queryParam = urlParams.get('q');
      if (queryParam && queryParam.trim().length >= SEARCH_CONFIG.minQueryLength) {
        document.getElementById('search-input').value = queryParam;
        setTimeout(() => performSearch(queryParam), 100);
      }
      
    } catch (error) {
      console.error('搜索初始化失败:', error);
      updateStats(`❌ 搜索功能初始化失败: ${error.message}`, true);
      showError('搜索功能暂时不可用，请刷新页面重试');
    }
  }

  // 执行搜索
  function performSearch(query) {
    if (!query || !isIndexReady) return;
    
    query = query.trim();
    
    // 验证查询长度
    if (query.length < SEARCH_CONFIG.minQueryLength) {
      if (query.length > 0) {
        showMessage(`请输入至少 ${SEARCH_CONFIG.minQueryLength} 个字符`);
      } else {
        showInitialMessage();
      }
      return;
    }
    
    // 显示搜索中状态
    showMessage('🔍 搜索中...', true);
    
    try {
      // 执行搜索
      const results = searchIndex.search(query, {
        fuzzy: SEARCH_CONFIG.fuzzy,
        prefix: true,
        combineWith: 'AND',
        boost: SEARCH_CONFIG.boost,
        weights: { fuzzy: 0.2, prefix: 0.8 }
      });
      
      // 限制结果数量
      const limitedResults = results.slice(0, SEARCH_CONFIG.maxResults);
      
      // 显示结果
      displayResults(limitedResults, query);
      
      // 更新统计信息
      updateStats(`找到 ${results.length} 个结果 "${query}"`);
      
      // 更新URL（不刷新页面）
      updateUrlWithQuery(query);
      
    } catch (error) {
      console.error('搜索失败:', error);
      showError('搜索过程中出现错误');
    }
  }

  // 显示搜索结果
  function displayResults(results, query) {
    const container = document.getElementById('search-results');
    
    if (results.length === 0) {
      container.innerHTML = `
        <div class="no-results">
          <h3>🔍 没有找到相关文章</h3>
          <p>没有找到包含 "<strong>${escapeHtml(query)}</strong>" 的文章</p>
          <div class="suggestions">
            <p>建议：</p>
            <ul>
              <li>尝试其他关键词或同义词</li>
              <li>减少搜索词数量</li>
              <li>检查拼写是否正确</li>
              <li>使用拼音搜索（如: "wangye" 搜索 "网页"）</li>
            </ul>
          </div>
        </div>
      `;
      return;
    }
    
    let html = `
      <div class="results-summary">
        <h3>📚 搜索结果</h3>
        <p>找到 <strong>${results.length}</strong> 个相关结果，搜索词: <strong>"${escapeHtml(query)}"</strong></p>
      </div>
      <div class="results-list">
    `;
    
    results.forEach((result, index) => {
      const post = postsData.find(p => p.id === result.id) || result;
      
      // 高亮匹配的文本
      const highlightedTitle = highlightMatches(post.title || '', query);
      const highlightedExcerpt = highlightMatches(
        post.excerpt || (post.content || '').substring(0, 150) + '...', 
        query
      );
      
      // 提取匹配片段用于预览
      const bestMatch = extractBestMatch(post.content || '', query);
      
      html += `
        <article class="search-result" data-score="${result.score || 0}">
          <div class="result-header">
            <span class="result-rank">#${index + 1}</span>
            <span class="result-score">匹配度: ${Math.round((result.score || 0) * 100)}%</span>
          </div>
          <h3 class="result-title">
            <a href="${post.url}" class="result-link">${highlightedTitle}</a>
          </h3>
          <div class="result-preview">
            ${bestMatch ? `<p class="best-match">✨ ${highlightMatches(bestMatch, query)}</p>` : ''}
            <p class="result-excerpt">${highlightedExcerpt}</p>
          </div>
          <div class="result-meta">
            <span class="result-date">📅 ${post.date || ''}</span>
            ${post.tags ? `<span class="result-tags">🏷️ ${formatTags(post.tags)}</span>` : ''}
            <a href="${post.url}" class="read-more">阅读全文 →</a>
          </div>
        </article>
      `;
    });
    
    html += `</div>`;
    container.innerHTML = html;
  }

  // 工具函数
  function highlightMatches(text, query) {
    if (!text || !query) return escapeHtml(text || '');
    
    const escapedText = escapeHtml(text);
    const escapedQuery = escapeHtml(query);
    
    // 将查询词拆分为多个关键词
    const keywords = escapedQuery.toLowerCase().split(/\s+/).filter(k => k.length > 0);
    
    if (keywords.length === 0) return escapedText;
    
    // 为每个关键词创建高亮
    let highlighted = escapedText;
    
    keywords.forEach(keyword => {
      if (keyword.length < 2) return;
      
      try {
        // 创建不区分大小写的正则表达式
        const regex = new RegExp(`(${keyword.replace(/[.*+?^${}()|[\]\\]/g, '\\$&')})`, 'gi');
        highlighted = highlighted.replace(regex, '<mark class="highlight">$1</mark>');
      } catch (e) {
        // 如果正则表达式创建失败，使用简单替换
        const lowerText = highlighted.toLowerCase();
        const lowerKeyword = keyword.toLowerCase();
        let position = lowerText.indexOf(lowerKeyword);
        
        while (position !== -1) {
          const before = highlighted.substring(0, position);
          const match = highlighted.substring(position, position + keyword.length);
          const after = highlighted.substring(position + keyword.length);
          
          highlighted = before + '<mark class="highlight">' + match + '</mark>' + after;
          position = lowerText.indexOf(lowerKeyword, position + keyword.length + 20); // +20 避免重复标记
        }
      }
    });
    
    return highlighted;
  }

  function extractBestMatch(content, query, length = 200) {
    if (!content || !query) return '';
    
    const lowerContent = content.toLowerCase();
    const lowerQuery = query.toLowerCase();
    
    // 尝试找到查询词在内容中的位置
    let bestPosition = -1;
    const keywords = lowerQuery.split(/\s+/).filter(k => k.length > 1);
    
    // 查找每个关键词的位置
    for (const keyword of keywords) {
      const position = lowerContent.indexOf(keyword);
      if (position !== -1 && (bestPosition === -1 || position < bestPosition)) {
        bestPosition = position;
      }
    }
    
    if (bestPosition === -1) {
      // 如果没有找到完整关键词，返回开头
      return content.substring(0, length) + '...';
    }
    
    // 提取包含关键词的片段
    const start = Math.max(0, bestPosition - Math.floor(length / 2));
    const end = Math.min(content.length, start + length);
    
    let excerpt = content.substring(start, end);
    if (start > 0) excerpt = '...' + excerpt;
    if (end < content.length) excerpt = excerpt + '...';
    
    return excerpt;
  }

  function formatTags(tags) {
    if (!tags) return '';
    
    if (Array.isArray(tags)) {
      return tags.map(tag => `<span class="tag">${escapeHtml(tag)}</span>`).join('');
    }
    
    if (typeof tags === 'string') {
      return tags.split(',').map(tag => 
        `<span class="tag">${escapeHtml(tag.trim())}</span>`
      ).join('');
    }
    
    return '';
  }

  function escapeHtml(text) {
    if (!text) return '';
    
    const div = document.createElement('div');
    div.textContent = text;
    return div.innerHTML;
  }

  function updateStats(message, isError = false) {
    const statsEl = document.getElementById('search-stats');
    if (!statsEl) return;
    
    statsEl.innerHTML = message;
    statsEl.className = isError ? 'search-stats error' : 'search-stats';
  }

  function showMessage(message, isLoading = false) {
    const container = document.getElementById('search-results');
    const className = isLoading ? 'loading-message' : 'info-message';
    
    container.innerHTML = `
      <div class="${className}">
        <p>${message}</p>
      </div>
    `;
  }

  function showInitialMessage() {
    const container = document.getElementById('search-results');
    container.innerHTML = `
      <div class="initial-message">
        <h3>💡 搜索提示</h3>
        <ul>
          <li>支持中文、英文、拼音搜索</li>
          <li>支持多个关键词（空格分隔）</li>
          <li>支持模糊匹配和拼写容错</li>
          <li>搜索结果按相关性排序</li>
        </ul>
      </div>
    `;
    
    // 绑定示例查询按钮
    document.querySelectorAll('.example-query').forEach(button => {
      button.addEventListener('click', function() {
        const query = this.getAttribute('data-query');
        document.getElementById('search-input').value = query;
        performSearch(query);
      });
    });
  }

  function showError(message) {
    const container = document.getElementById('search-results');
    container.innerHTML = `
      <div class="error-message">
        <h3>⚠️ 出错了</h3>
        <p>${message}</p>
        <button onclick="window.location.reload()">刷新页面</button>
      </div>
    `;
  }

  function updateUrlWithQuery(query) {
    const url = new URL(window.location);
    
    if (query && query.length >= SEARCH_CONFIG.minQueryLength) {
      url.searchParams.set('q', query);
    } else {
      url.searchParams.delete('q');
    }
    
    window.history.replaceState({}, '', url);
  }



  // 页面初始化
  document.addEventListener('DOMContentLoaded', function() {
    // 初始化搜索
    initSearch();
    
    const searchInput = document.getElementById('search-input');
    const clearButton = document.getElementById('clear-search');
    
    // 搜索输入事件
    const debouncedSearch = debounce(function(e) {
      performSearch(e.target.value.trim());
    }, SEARCH_CONFIG.debounceTime);
    
    searchInput.addEventListener('input', debouncedSearch);
    
    // Enter键搜索
    searchInput.addEventListener('keypress', function(e) {
      if (e.key === 'Enter') {
        e.preventDefault();
        performSearch(this.value.trim());
      }
    });
    
    // 清空搜索
    clearButton.addEventListener('click', function() {
      searchInput.value = '';
      searchInput.focus();
      showInitialMessage();
      updateStats(`已索引 ${postsData.length} 篇文章`);
      updateUrlWithQuery('');
    });
    
    // 自动聚焦
    setTimeout(() => {
      searchInput.focus();
      
      // 如果有URL参数，填充输入框
      const urlParams = new URLSearchParams(window.location.search);
      const queryParam = urlParams.get('q');
      if (queryParam) {
        searchInput.value = queryParam;
      }
    }, 100);
    
    // 处理浏览器前进/后退
    window.addEventListener('popstate', function() {
      const urlParams = new URLSearchParams(window.location.search);
      const queryParam = urlParams.get('q');
      
      if (queryParam && isIndexReady) {
        searchInput.value = queryParam;
        performSearch(queryParam);
      } else if (searchInput.value) {
        searchInput.value = '';
        showInitialMessage();
      }
    });
  });

</script>



<style>
  /* 搜索页面样式 */
  .search-container {
    max-width: 900px;
    margin: 0 auto;
    padding: 2rem 1rem;
  }
  
  .search-header {
    margin-bottom: 2rem;
  }
  
  .search-header h1 {
    margin-bottom: 1.5rem;
    color: #333;
  }
  
  .search-box {
    position: relative;
    margin-bottom: 1rem;
  }
  
  #search-input {
    width: 100%;
    padding: 1rem 0rem 1rem 1.5rem;
    font-size: 1.1rem;
    border: 2px solid #ddd;
    border-radius: 0px;
    outline: none;
    transition: all 0.3s ease;
    background: #f8f9fa;
  }
  
  #search-input:focus {
    border-color: #007bff;
    background: white;
    box-shadow: 0 0 0 3px rgba(0, 123, 255, 0.1);
  }
  
  .search-actions {
    position: absolute;
    right: 0rem;
    top: 50%;
    transform: translateY(-50%);
  }
  
  #clear-search {
    background: none;
    border: none;
    color: #999;
    font-size: 1.2rem;
    cursor: pointer;
    padding: 0.5rem;
    border-radius: 50%;
  }
  
  #clear-search:hover {
    background: #f0f0f0;
    color: #666;
  }
  
  .search-stats {
    color: #666;
    font-size: 0.9rem;
    min-height: 1.5rem;
  }
  
  .search-stats.error {
    color: #dc3545;
  }
  
  /* 搜索结果样式 */
  .search-results {
    min-height: 300px;
  }
  
  .initial-message,
  .loading-message,
  .info-message,
  .no-results,
  .error-message {
    background: #f8f9fa;
    border-radius: 12px;
    padding: 2rem;
    text-align: center;
    margin: 2rem 0;
  }
  
  .initial-message h3,
  .no-results h3,
  .error-message h3 {
    margin-top: 0;
    color: #333;
  }
  
  .initial-message ul,
  .no-results .suggestions ul {
    text-align: left;
    display: inline-block;
    padding-left: 1.5rem;
    color: #666;
	 margin-left:0px;
  }
  
  .initial-message li,
  .no-results .suggestions li {
    margin-bottom: 0.5rem;
  }
  
  .example-queries {
    margin-top: 1.5rem;
  }
  
  .example-queries p {
    margin-bottom: 0.5rem;
    color: #666;
  }
  
  .example-query {
    background: #e9ecef;
    border: none;
    padding: 0.5rem 1rem;
    margin: 0 0.5rem 0.5rem 0;
    border-radius: 20px;
    color: #495057;
    cursor: pointer;
    transition: all 0.2s ease;
    font-size: 0.9rem;
  }
  
  .example-query:hover {
    background: #dee2e6;
    color: #212529;
  }
  
  /* 搜索结果列表 */
  .results-summary {
    margin-bottom: 2rem;
    padding-bottom: 1rem;
    border-bottom: 1px solid #eee;
  }
  
  .results-summary h3 {
    margin: 0 0 0.5rem 0;
    color: #333;
  }
  
  .results-summary p {
    color: #666;
    margin: 0;
  }
  
  .search-result {
    background: white;
    border: 1px solid #e9ecef;
    border-radius: 12px;
    padding: 1.5rem;
    margin-bottom: 1.5rem;
    transition: all 0.3s ease;
  }
  
  .search-result:hover {
    border-color: #007bff;
    box-shadow: 0 5px 15px rgba(0, 123, 255, 0.1);
    transform: translateY(-2px);
  }
  
  .result-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 0.5rem;
    font-size: 0.85rem;
    color: #666;
  }
  
  .result-rank {
    background: #007bff;
    color: white;
    padding: 0.2rem 0.6rem;
    border-radius: 10px;
    font-weight: bold;
  }
  
  .result-score {
    opacity: 0.7;
  }
  
  .result-title {
    margin: 0 0 1rem 0;
  }
  
  .result-link {
    color: #333;
    text-decoration: none;
    font-size: 1.3rem;
    font-weight: 600;
  }
  
  .result-link:hover {
    color: #007bff;
  }
  
  .result-preview {
    margin-bottom: 1rem;
    color: #555;
    line-height: 1.6;
  }
  
  .best-match {
    background: #fff8e1;
    padding: 0.8rem;
    border-radius: 8px;
    border-left: 3px solid #ffc107;
    margin-bottom: 0.8rem;
    font-style: italic;
  }
  
  .result-excerpt {
    margin: 0;
  }
  
  .highlight {
    background: #fff8c5;
    padding: 0.1rem 0.3rem;
    border-radius: 3px;
    color: #333;
    font-weight: 500;
  }
  
  .result-meta {
    display: flex;
    align-items: center;
    flex-wrap: wrap;
    gap: 1rem;
    font-size: 0.85rem;
    color: #6c757d;
  }
  
  .result-date {
    display: flex;
    align-items: center;
    gap: 0.3rem;
  }
  
  .result-tags {
    display: flex;
    align-items: center;
    gap: 0.5rem;
  }
  
  .tag {
    background: #e7f1ff;
    color: #0056b3;
    padding: 0.2rem 0.6rem;
    border-radius: 12px;
    font-size: 0.8rem;
  }
  
  .read-more {
    margin-left: auto;
    color: #007bff;
    text-decoration: none;
    font-weight: 500;
  }
  
  .read-more:hover {
    text-decoration: underline;
  }
  
  /* 响应式设计 */
  @media (max-width: 768px) {
    .search-container {
      padding: 1rem;
    }
    
    .search-header h1 {
      font-size: 1.8rem;
    }
    
    #search-input {
      padding: 0.8rem 0rem 0.8rem 0rem;
      font-size: 1rem;
    }
    
    .search-result {
      padding: 1rem;
    }
    
    .result-link {
      font-size: 1.1rem;
    }
    
    .result-meta {
      gap: 0.5rem;
    }
    
    .read-more {
      margin-left: 0;
      margin-top: 0.5rem;
      width: 100%;
      text-align: right;
    }
  }
  
  @media (max-width: 480px) {
    .initial-message,
    .no-results,
    .error-message {
      padding: 1.5rem 1rem;
    }
    
    .result-header {
      flex-direction: column;
      align-items: flex-start;
      gap: 0.5rem;
    }
  }
</style>