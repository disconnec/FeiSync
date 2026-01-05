<template>
  <section class="page-shell">
    <div v-if="toasts.length" class="toast-stack">
      <div v-for="toast in toasts" :key="toast.id" class="toast-item">
        {{ toast.text }}
      </div>
    </div>
    <article class="card">
      <div class="header-block">
        <div class="header-bar">
          <div class="title-block">
            <h2>日志配置</h2>
          </div>
          <div class="toggle-inline">
            <span>启用文件日志</span>
            <button
              type="button"
              class="toggle-switch"
              :class="{ active: form.enabled }"
              @click="toggleLogging"
              role="switch"
              :aria-checked="form.enabled"
            >
              <span class="sr-only">启用文件日志</span>
            </button>
          </div>
        </div>
        <p v-if="!form.enabled" class="disabled-note">当前日志仅输出到 stdout，启用后即可设置文件目录及容量限制。</p>
      </div>
      <form v-if="form.enabled" class="config-form" @submit.prevent="saveConfig">
        <section class="form-section">
          <div class="form-group">
            <label class="form-label">日志目录</label>
            <div class="input-group">
              <div class="input-wrapper">
                <input type="text" v-model="form.directory" placeholder="选择日志文件存储目录" />
              </div>
              <button class="btn secondary" type="button" @click="pickDirectory">浏览</button>
            </div>
          </div>
          <div class="form-group compact">
            <label class="form-label">容量上限</label>
            <div class="number-input-wrapper">
              <input type="number" v-model.number="form.max_size_mb" min="5" max="2048" />
            </div>
          </div>
        </section>
        <div class="button-group">
          <button class="btn secondary" type="button" @click="resetForm" :disabled="saving || loadingConfig">
            还原
          </button>
          <button class="btn secondary" type="button" @click="openDirectory" :disabled="!form.directory">打开目录</button>
          <button class="btn primary" type="submit" :disabled="saving || loadingConfig">
            <span v-if="saving" class="spinner small"></span>
            <span>{{ saving ? '保存中...' : '保存配置' }}</span>
          </button>
        </div>
      </form>
    </article>

    <article class="card">
      <header class="logs-head">
        <div>
          <h2>API 调用日志</h2>
          <p>默认展示最新 {{ filters.limit }} 条，可通过筛选条件获取更多信息。</p>
        </div>
        <div class="log-actions">
          <button class="btn secondary" type="button" @click="loadLogs" :disabled="loadingLogs">
            <span v-if="loadingLogs" class="spinner small"></span>
            <span>{{ loadingLogs ? '加载中...' : '刷新' }}</span>
          </button>
        </div>
      </header>
      <section class="filters">
        <label>
          命令
          <input v-model="filters.command" placeholder="模糊匹配，如 list_tenants" />
        </label>
        <label class="compact">
          状态
          <select v-model="filters.status">
            <option value="">全部</option>
            <option value="success">成功</option>
            <option value="error">错误</option>
          </select>
        </label>
        <label class="compact">
          条数
          <input type="number" v-model.number="filters.limit" min="50" max="2000" />
        </label>
        <div class="filter-actions">
          <button class="btn secondary" type="button" @click="clearLogs" :disabled="!logs.length || clearing">
            <span v-if="clearing" class="spinner small"></span>
            <span>{{ clearing ? '清空中...' : '清空列表' }}</span>
          </button>
          <button class="btn secondary" type="button" @click="resetFilters">清空条件</button>
          <button class="btn primary" type="button" @click="loadLogs" :disabled="loadingLogs">查询</button>
        </div>
      </section>
      <div class="log-table-wrapper">
        <table class="log-table">
          <thead>
            <tr>
              <th>时间</th>
              <th>范围</th>
              <th>命令</th>
              <th>状态</th>
              <th>耗时</th>
              <th>信息</th>
            </tr>
          </thead>
          <tbody>
            <template v-for="entry in logs" :key="entry.id">
              <tr @click="selectEntry(entry)" :class="{ active: selectedLog?.id === entry.id }">
                <td>{{ formatTime(entry.timestamp) }}</td>
                <td>{{ entry.scope }}</td>
                <td>{{ entry.command }}</td>
                <td>
                  <span :class="['status-chip', entry.status === 'success' ? 'ok' : 'err']">
                    {{ entry.status === 'success' ? '成功' : '错误' }}
                  </span>
                </td>
                <td>{{ entry.duration_ms ? entry.duration_ms + ' ms' : '-' }}</td>
                <td>{{ entry.message || '—' }}</td>
              </tr>
              <tr v-if="selectedLog?.id === entry.id" class="detail-row">
                <td colspan="6">
                  <div class="row-detail">
                    <strong>请求/响应详情</strong>
                    <pre v-if="entry.meta">{{ prettify(entry.meta) }}</pre>
                    <p v-else class="detail-empty">暂无详细信息</p>
                  </div>
                </td>
              </tr>
            </template>
            <tr v-if="!logs.length">
              <td colspan="6" class="empty">暂无日志</td>
            </tr>
          </tbody>
        </table>
      </div>
    </article>
  </section>
</template>

<script setup lang="ts">
import { onMounted, reactive, ref } from 'vue';
import { invoke } from '@tauri-apps/api/core';

interface ApiLogEntry {
  id: string;
  command: string;
  scope: string;
  status: 'success' | 'error';
  message?: string | null;
  duration_ms?: number | null;
  timestamp: string;
  meta?: Record<string, unknown> | null;
}

interface LogConfig {
  enabled: boolean;
  directory: string | null;
  max_size_mb: number;
}

const form = reactive<LogConfig>({
  enabled: false,
  directory: null,
  max_size_mb: 100
});
const DEFAULT_MAX_SIZE = 100;
const originalConfig = ref<LogConfig | null>(null);
const loadingConfig = ref(false);
const saving = ref(false);
const logs = ref<ApiLogEntry[]>([]);
const loadingLogs = ref(false);
const filters = reactive({
  command: '',
  status: '',
  limit: 200
});
const selectedLog = ref<ApiLogEntry | null>(null);
const toasts = reactive<{ id: number; text: string }[]>([]);
const clearing = ref(false);

const addToast = (text: string) => {
  const id = Date.now() + Math.random();
  toasts.push({ id, text });
  setTimeout(() => {
    const idx = toasts.findIndex((item) => item.id === id);
    if (idx >= 0) {
      toasts.splice(idx, 1);
    }
  }, 2200);
};

const loadConfig = async () => {
  loadingConfig.value = true;
  try {
    const config = await invoke<LogConfig>('get_log_config');
    form.enabled = config.enabled;
    form.directory = config.directory || null;
    const resolved = Number(config.max_size_mb);
    form.max_size_mb = Number.isFinite(resolved) && resolved > 0 ? resolved : DEFAULT_MAX_SIZE;
    originalConfig.value = { ...form };
  } catch (error) {
    addToast(error instanceof Error ? error.message : String(error));
  } finally {
    loadingConfig.value = false;
  }
};

const saveConfig = async () => {
  saving.value = true;
  try {
    const payload = {
      enabled: form.enabled,
      directory: form.enabled ? form.directory : null,
      max_size_mb: form.max_size_mb
    };
    const config = await invoke<LogConfig>('update_log_config', { payload });
    form.enabled = config.enabled;
    form.directory = config.directory || null;
    const resolved = Number(config.max_size_mb);
    form.max_size_mb = Number.isFinite(resolved) && resolved > 0 ? resolved : DEFAULT_MAX_SIZE;
    originalConfig.value = { ...form };
    addToast('配置已保存');
  } catch (error) {
    addToast(error instanceof Error ? error.message : String(error));
  } finally {
    saving.value = false;
  }
};

const pickDirectory = async () => {
  if (!form.enabled) return;
  try {
    const dir = await invoke<string | null>('pick_directory_dialog');
    if (dir) {
      form.directory = dir;
    }
  } catch (error) {
    addToast('选择目录失败');
  }
};

const toggleLogging = () => {
  form.enabled = !form.enabled;
};

const openDirectory = async () => {
  if (!form.directory) return;
  try {
    await invoke('reveal_local_path', { path: form.directory });
  } catch (error) {
    addToast('打开目录失败');
  }
};

const resetForm = () => {
  if (!originalConfig.value) return;
  Object.assign(form, originalConfig.value);
};

const loadLogs = async () => {
  loadingLogs.value = true;
  selectedLog.value = null;
  try {
    const payload: Record<string, unknown> = {
      command: filters.command || undefined,
      status: filters.status || undefined,
      limit: filters.limit
    };
    const cleaned = Object.fromEntries(
      Object.entries(payload).filter(([, value]) => value !== undefined && value !== '')
    );
    logs.value = await invoke<ApiLogEntry[]>('list_api_logs', { payload: cleaned });
  } catch (error) {
    addToast(error instanceof Error ? error.message : String(error));
  } finally {
    loadingLogs.value = false;
  }
};

const clearLogs = () => {
  if (!logs.value.length) return;
  clearing.value = true;
  logs.value = [];
  selectedLog.value = null;
  setTimeout(() => {
    clearing.value = false;
  }, 150);
};

const selectEntry = (entry: ApiLogEntry) => {
  if (selectedLog.value?.id === entry.id) {
    selectedLog.value = null;
  } else {
    selectedLog.value = entry;
  }
};

const resetFilters = () => {
  filters.command = '';
  filters.status = '';
  filters.limit = 200;
};

const formatTime = (iso: string) => {
  const date = new Date(iso);
  if (Number.isNaN(date.getTime())) return iso;
  return date.toLocaleString();
};

const normalizeMeta = (value: unknown): unknown => {
  if (typeof value === 'string') {
    const trimmed = value.trim();
    if ((trimmed.startsWith('{') && trimmed.endsWith('}')) || (trimmed.startsWith('[') && trimmed.endsWith(']'))) {
      try {
        return normalizeMeta(JSON.parse(trimmed));
      } catch {
        return value;
      }
    }
    return value;
  }
  if (Array.isArray(value)) {
    return value.map((item) => normalizeMeta(item));
  }
  if (value && typeof value === 'object') {
    return Object.fromEntries(
      Object.entries(value as Record<string, unknown>).map(([key, val]) => [key, normalizeMeta(val)])
    );
  }
  return value;
};

const prettify = (meta?: Record<string, unknown> | null) => {
  if (!meta) return '';
  return JSON.stringify(normalizeMeta(meta), null, 2);
};

onMounted(async () => {
  await loadConfig();
  await loadLogs();
});
</script>

<style scoped>
.page-shell {
  display: flex;
  flex-direction: column;
  gap: 1.25rem;
  background: #fff;
  min-height: 100%;
  padding: 1.75rem 1.5rem;
}

.card {
  background: white;
  border-radius: 1rem;
  padding: 1.5rem;
  border: 1px solid #e2eaf4;
  box-shadow: 0 8px 20px rgba(15, 23, 42, 0.08);
}

.card header h2 {
  margin-bottom: 0.2rem;
}

.header-block {
  margin-bottom: 1rem;
  padding-bottom: 1rem;
}

.header-bar {
  display: flex;
  justify-content: space-between;
  gap: 1rem;
  align-items: center;
}

.toggle-inline {
  display: inline-flex;
  align-items: center;
  gap: 0.6rem;
  font-weight: 600;
}

.toggle-switch {
  width: 48px;
  height: 26px;
  border-radius: 999px;
  background: #cbd5e1;
  border: none;
  cursor: pointer;
  position: relative;
  transition: background 0.25s ease;
}

.toggle-switch::after {
  content: '';
  position: absolute;
  top: 3px;
  left: 3px;
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: white;
  box-shadow: 0 2px 6px rgba(15, 23, 42, 0.2);
  transition: transform 0.25s ease;
}

.toggle-switch.active {
  background: #3b82f6;
}

.toggle-switch.active::after {
  transform: translateX(20px);
}

.disabled-note {
  margin-top: 0.85rem;
  background: #fef3c7;
  border: 1px solid #facc15;
  border-radius: 0.65rem;
  padding: 0.75rem 1rem;
  font-size: 0.9rem;
  color: #92400e;
}

.config-form {
  display: flex;
  flex-direction: column;
  gap: 1.4rem;
}

.form-section {
  display: flex;
  gap: 1.25rem;
  flex-wrap: wrap;
}

.form-group {
  flex: 1 1 360px;
  display: flex;
  flex-direction: column;
  gap: 0.45rem;
}

.form-group.compact {
  flex: 0 0 220px;
}

.form-label {
  font-size: 0.95rem;
  font-weight: 600;
  color: #334155;
  display: inline-flex;
  align-items: center;
  gap: 0.35rem;
}

.form-hint {
  font-size: 0.85rem;
  color: #64748b;
  display: flex;
  align-items: center;
  gap: 0.35rem;
}

.form-hint::before {
  content: 'ℹ';
  width: 18px;
  height: 18px;
  border-radius: 50%;
  background: #e0f2fe;
  color: #0369a1;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  font-size: 0.75rem;
  font-weight: 700;
}

.input-group {
  display: flex;
  gap: 0.65rem;
}

.input-wrapper {
  flex: 1;
}

.form-group input {
  border: 2px solid #e2e8f0;
  border-radius: 0.65rem;
  padding: 0.65rem 0.85rem;
  transition: border-color 0.2s ease, box-shadow 0.2s ease;
}

.form-group input:focus {
  border-color: #3b82f6;
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.15);
}

.number-input-wrapper {
  position: relative;
}

.number-input-wrapper::after {
  content: 'MB';
  position: absolute;
  right: 0.9rem;
  top: 50%;
  transform: translateY(-50%);
  color: #94a3b8;
  font-size: 0.85rem;
  font-weight: 600;
  pointer-events: none;
}

.number-input-wrapper input {
  padding-right: 2.5rem;
}

.button-group {
  display: flex;
  gap: 0.75rem;
  flex-wrap: wrap;
  align-items: center;
  padding-top: 1rem;
  border-top: 1px solid #e2eaf4;
}

.button-group .btn.primary {
  margin-left: auto;
}

.button-group .btn {
  min-width: 140px;
  justify-content: center;
}

.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  border: 0;
}

.config-form label,
.filters label {
  display: flex;
  flex-direction: column;
  gap: 0.35rem;
  font-size: 0.95rem;
}

@media (max-width: 820px) {
  .form-group,
  .form-group.compact,
  .filters label,
  .filters label.compact {
    flex: 1 1 100%;
  }

  .input-group {
    flex-direction: column;
  }

  .header-bar {
    flex-direction: column;
    align-items: flex-start;
  }

  .toggle-inline {
    margin-top: 0.5rem;
  }

  .button-group {
    flex-direction: column;
    align-items: stretch;
  }

  .button-group .btn {
    width: 100%;
  }

  .button-group .btn.primary {
    margin-left: 0;
  }
}

input,
select {
  width: 100%;
  padding: 0.55rem 0.65rem;
  border: 1px solid #cbd5f5;
  border-radius: 0.65rem;
  font-size: 0.95rem;
  min-height: 2.6rem;
  height: 2.6rem;
}

.logs-head {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 0.75rem;
}

.filters {
  display: flex;
  flex-wrap: wrap;
  gap: 0.75rem;
  margin: 1rem 0;
}

.filters label {
  flex: 1 1 240px;
}

.filters label.compact {
  flex: 0 0 150px;
}

.filter-actions {
  display: flex;
  align-items: flex-end;
  gap: 0.5rem;
}

.log-table-wrapper {
  overflow-x: auto;
}

.log-table {
  width: 100%;
  border-collapse: collapse;
}

.log-table th,
.log-table td {
  padding: 0.65rem;
  border-bottom: 1px solid #f0f4ff;
  text-align: left;
}

.log-table tbody tr {
  cursor: pointer;
}

.log-table tbody tr:hover,
.log-table tbody tr.active {
  background: #f6f9ff;
}

.detail-row td {
  background: #f6f9ff;
  padding: 0;
}

.row-detail {
  padding: 0.85rem 1rem 1.1rem;
  display: flex;
  flex-direction: column;
  gap: 0.7rem;
}

.row-detail strong {
  font-size: 0.9rem;
}

.status-chip {
  display: inline-flex;
  align-items: center;
  padding: 0.1rem 0.6rem;
  border-radius: 999px;
  font-size: 0.8rem;
}

.status-chip.ok {
  background: #dcfce7;
  color: #166534;
}

.status-chip.err {
  background: #fee2e2;
  color: #991b1b;
}

.label {
  font-size: 0.8rem;
  color: #64748b;
}

pre {
  background: #0f172a;
  color: #e2e8f0;
  padding: 0.75rem;
  border-radius: 0.75rem;
  overflow: auto;
  font-size: 0.85rem;
}

.detail-empty {
  color: #94a3b8;
  font-size: 0.9rem;
}

.empty {
  text-align: center;
  padding: 0.75rem;
  color: #8c96b6;
}

.toast-stack {
  position: fixed;
  top: 1.25rem;
  right: 1.5rem;
  display: flex;
  flex-direction: column;
  gap: 0.45rem;
  z-index: 9500;
}

.toast-item {
  background: rgba(15, 23, 42, 0.9);
  color: #fff;
  padding: 0.5rem 0.85rem;
  border-radius: 0.65rem;
  box-shadow: 0 10px 20px rgba(15, 23, 42, 0.35);
  font-size: 0.9rem;
}

.spinner {
  width: 14px;
  height: 14px;
  border-radius: 50%;
  border: 2px solid #1d7bff;
  border-top-color: transparent;
  animation: spin 0.8s linear infinite;
}

.spinner.small {
  width: 12px;
  height: 12px;
}

@keyframes spin {
  from {
    transform: rotate(0deg);
  }
  to {
    transform: rotate(360deg);
  }
}
</style>
