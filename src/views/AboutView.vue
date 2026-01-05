<template>
  <section class="page-shell">
    <div class="container">
      <header class="header">
        <div class="title-row">
          <h1>FeiSync</h1>
          <span class="badge version">v1.0.0</span>
        </div>
        <p class="subtitle">
          基于 Tauri + Vue3 打造的高性能飞书/Lark 云空间管理客户端。
          <br>
          <span class="tech-stack">Rust 核心驱动 · Pinia 状态管理 · 跨平台支持</span>
        </p>
      </header>

      <div class="features-grid">
        <div class="feature-card">
          <h3>多租户智能聚合</h3>
          <p>支持配置多个企业实例，通过 app_id/secret 分组筛选。智能监控容量，空间不足时自动切换实例，无需关心底层租户逻辑。</p>
        </div>

        <div class="feature-card">
          <h3>极致文件管理</h3>
          <p>支持列表/图标双模式、面包屑导航与全局搜索。提供批量操作、拖拽上传/移动，带来原生级的文件浏览体验。</p>
        </div>

        <div class="feature-card">
          <h3>可靠传输中心</h3>
          <p>可视化管理上传/下载队列。支持断点续传、失败重试、暂停/恢复，实时监控传输速度与剩余时间，大文件传输无忧。</p>
        </div>

        <div class="feature-card">
          <h3>自动化与扩展</h3>
          <p>内置 Cron 定时同步任务。开放 API 供第三方调用，支持单文件 250MB 上传，单一免费实例可扩容至 100GB。</p>
        </div>
      </div>
    </div>
  </section>
</template>

<script setup lang="ts">
// 逻辑层保持精简，如果有动态数据可以在此定义
</script>

<style scoped>
/* 基础容器 */
.page-shell {
  display: flex;
  justify-content: center;
  background: #f8fafc; /* 更柔和的背景色 */
  min-height: 100%;
  padding: 2rem;
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
}

.container {
  max-width: 900px;
  width: 100%;
  display: flex;
  flex-direction: column;
  gap: 2rem;
}

/* 头部样式 */
.header {
  text-align: center;
  margin-bottom: 1rem;
}

.title-row {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 1rem;
  margin-bottom: 0.5rem;
}

h1 {
  font-size: 2.5rem;
  font-weight: 700;
  color: #1e293b;
  margin: 0;
  letter-spacing: -0.025em;
}

.badge {
  background: #e0f2fe;
  color: #0284c7;
  padding: 0.25rem 0.75rem;
  border-radius: 999px;
  font-size: 0.875rem;
  font-weight: 600;
}

.subtitle {
  color: #64748b;
  line-height: 1.6;
  font-size: 1.1rem;
}

.tech-stack {
  font-size: 0.9rem;
  color: #94a3b8;
  margin-top: 0.5rem;
  display: inline-block;
}

/* 特性网格布局 */
.features-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
  gap: 1.5rem;
}

/* 特性卡片 */
.feature-card {
  background: #fff;
  border: 1px solid #e2e8f0;
  border-radius: 1rem;
  padding: 1.5rem;
  transition: all 0.3s ease;
  box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.05);
}

.feature-card:hover {
  transform: translateY(-4px);
  box-shadow: 0 10px 15px -3px rgba(0, 0, 0, 0.1);
  border-color: #cbd5e1;
}

.icon {
  font-size: 2rem;
  margin-bottom: 1rem;
}

.feature-card h3 {
  font-size: 1.25rem;
  font-weight: 600;
  color: #334155;
  margin: 0 0 0.75rem 0;
}

.feature-card p {
  color: #64748b;
  font-size: 0.95rem;
  line-height: 1.6;
  margin: 0;
}
</style>