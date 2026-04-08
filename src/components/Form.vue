<template>
  <!-- 模板部分与之前完全相同，省略重复 -->
  <div class="screenshot-plugin">
    <el-card class="config-card" shadow="never">
      <template #header>
        <div class="card-header">
          <span>📸 表格长图生成器</span>
        </div>
      </template>

      <el-form label-width="120px" label-position="left">
        <el-form-item label="不显示的字段">
          <el-select
            v-model="excludedFields"
            multiple
            filterable
            collapse-tags
            collapse-tags-tooltip
            placeholder="选择要隐藏的字段（可多选）"
            style="width: 100%"
          >
            <el-option
              v-for="field in allFields"
              :key="field.id"
              :label="field.name"
              :value="field.name"
            />
          </el-select>
          <div class="field-hint">
            当前显示 <strong>{{ displayedFields.length }}</strong> 个字段（共 {{ allFields.length }} 个）
            <el-button link type="primary" @click="excludedFields = []">清空排除</el-button>
            <el-button link type="danger" @click="excludedFields = allFields.map(f => f.name)">全部排除</el-button>
            <el-button link type="success" @click="syncHiddenFields" style="margin-left: 8px;">同步视图隐藏字段</el-button>
          </div>
        </el-form-item>

        <el-divider content-position="left">表格样式</el-divider>
        <el-row :gutter="20">
          <el-col :span="12">
            <el-form-item label="边框颜色">
              <el-color-picker v-model="tableStyle.borderColor" size="small" />
            </el-form-item>
          </el-col>
          <el-col :span="12">
            <el-form-item label="表头背景">
              <el-color-picker v-model="tableStyle.headerBgColor" size="small" />
            </el-form-item>
          </el-col>
          <el-col :span="12">
            <el-form-item label="表头文字色">
              <el-color-picker v-model="tableStyle.headerTextColor" size="small" />
            </el-form-item>
          </el-col>
          <el-col :span="12">
            <el-form-item label="文字颜色">
              <el-color-picker v-model="tableStyle.textColor" size="small" />
            </el-form-item>
          </el-col>
          <el-col :span="12">
            <el-form-item label="行背景色">
              <el-color-picker v-model="tableStyle.rowBgColor" size="small" />
            </el-form-item>
          </el-col>
          <el-col :span="12">
            <el-form-item label="交替行背景">
              <el-color-picker v-model="tableStyle.altRowBgColor" size="small" />
            </el-form-item>
          </el-col>
          <el-col :span="12">
            <el-form-item label="字体大小(px)">
              <el-input-number v-model="tableStyle.fontSize" :min="10" :max="24" size="small" />
            </el-form-item>
          </el-col>
          <el-col :span="12">
            <el-form-item label="行高(px)">
              <el-input-number v-model="tableStyle.lineHeight" :min="30" :max="60" size="small" />
            </el-form-item>
          </el-col>
        </el-row>
      </el-form>

      <div class="action-buttons">
        <el-button type="primary" :loading="downloadLoading" @click="generateAndSave">
          {{ downloadLoading ? '生成中...' : '保存为表格长图' }}
        </el-button>
        <el-button @click="manualRefreshPreview" :loading="previewLoading">
          刷新预览
        </el-button>
      </div>
    </el-card>

    <el-card class="preview-card" shadow="never">
      <template #header>
        <div class="card-header">
          <span>📷 实时预览</span>
          <div class="preview-tools">
            <el-tag size="small" type="info" style="margin-right: 8px;">自动更新</el-tag>
            <div class="zoom-controls">
              <span class="zoom-label">缩放:</span>
              <el-slider v-model="zoomPercent" :min="30" :max="300" :step="5" class="zoom-slider" />
              <span class="zoom-percent">{{ zoomPercent }}%</span>
              <el-button size="small" @click="resetZoom">重置</el-button>
            </div>
          </div>
        </div>
      </template>
      <div class="preview-container" v-loading="previewLoading">
        <img v-if="previewImageUrl" :src="previewImageUrl" class="preview-image" :style="imageZoomStyle" alt="表格预览" />
        <div v-else class="preview-placeholder">
          <el-icon><Picture /></el-icon>
          <span>暂无预览，请选中记录并确保至少显示一个字段</span>
        </div>
      </div>
    </el-card>

    <div ref="screenshotContainer" class="screenshot-container" :style="containerStyle">
      <table class="data-table" :style="tableStyleInline">
        <thead>
          <tr><th v-for="fieldName in displayedFields" :key="fieldName">{{ fieldName }}</th></tr>
        </thead>
        <tbody>
          <tr v-for="(record, idx) in selectedRecordsData" :key="record.recordId || idx">
            <td v-for="fieldName in displayedFields" :key="fieldName">{{ getFieldValue(record, fieldName) }}</td>
          </tr>
        </tbody>
      </table>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, computed, watch, nextTick, onUnmounted } from 'vue';
import html2canvas from 'html2canvas';
import { bitable } from '@lark-base-open/js-sdk';
import { ElMessage } from 'element-plus';
import { Picture } from '@element-plus/icons-vue';

// ---------- 数据 ----------
const allFields = ref([]);
const excludedFields = ref([]);
const selectedRecordsData = ref([]);
const screenshotContainer = ref(null);
const fieldNameToId = ref({});

const downloadLoading = ref(false);
const previewLoading = ref(false);
const previewImageUrl = ref('');
const zoomPercent = ref(100);

const defaultTableStyle = {
  borderColor: '#e0e0e0',
  headerBgColor: '#f5f7fa',
  headerTextColor: '#333333',
  rowBgColor: '#ffffff',
  altRowBgColor: '#fafafa',
  textColor: '#555555',
  fontSize: 14,
  lineHeight: 40,
};
const tableStyle = ref({ ...defaultTableStyle });

const displayedFields = computed(() => allFields.value.map(f => f.name).filter(name => !excludedFields.value.includes(name)));

const tableStyleInline = computed(() => ({
  borderCollapse: 'collapse',
  width: '100%',
  fontSize: `${tableStyle.value.fontSize}px`,
  fontFamily: '-apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif',
}));

const containerStyle = computed(() => ({
  position: 'fixed',
  left: '-9999px',
  top: 0,
}));

const imageZoomStyle = computed(() => ({
  width: `${zoomPercent.value}%`,
  height: 'auto',
}));

// ---------- 字段类型常量 ----------
const FieldType = {
  TEXT: 1, NUMBER: 2, SINGLE_SELECT: 3, MULTI_SELECT: 4, DATE: 5,
  CHECKBOX: 7, USER: 11, URL: 13, PHONE: 14, EMAIL: 15, IMAGE: 16,
  ATTACHMENT: 17, LINK: 18, FORMULA: 20, AUTO_NUMBER: 21, LOCATION: 22,
};

// ---------- 辅助函数 ----------
const formatDateValue = (value) => {
  if (value === undefined || value === null) return '';
  let timestamp = value;
  if (typeof timestamp === 'string') timestamp = parseInt(timestamp, 10);
  if (typeof timestamp !== 'number' || isNaN(timestamp)) return String(value);
  if (timestamp < 10000000000) timestamp *= 1000;
  const date = new Date(timestamp);
  if (isNaN(date.getTime())) return String(value);
  const pad = (n) => n.toString().padStart(2, '0');
  return `${date.getFullYear()}-${pad(date.getMonth() + 1)}-${pad(date.getDate())} ${pad(date.getHours())}:${pad(date.getMinutes())}:${pad(date.getSeconds())}`;
};

const formatCheckboxValue = (value) => (value === true ? '是' : value === false ? '否' : '');

const formatFieldValue = (value, fieldType) => {
  if (value === undefined || value === null) return '';
  if (fieldType === FieldType.DATE) return formatDateValue(value);
  if (fieldType === FieldType.CHECKBOX) return formatCheckboxValue(value);
  if (fieldType === FieldType.NUMBER) {
    if (typeof value === 'number') return Number.isInteger(value) ? value.toString() : value.toFixed(2);
    return String(value);
  }
  if (fieldType === FieldType.URL) {
    if (typeof value === 'object' && value !== null) return value.text || value.link || JSON.stringify(value);
    return String(value);
  }
  if (Array.isArray(value)) {
    return value.map(item => {
      if (typeof item === 'object' && item !== null) return item.name || item.text || item.title || item.id || JSON.stringify(item);
      return String(item);
    }).join(', ');
  }
  if (typeof value === 'object') {
    try {
      if (value.name) return value.name;
      if (value.text) return value.text;
      if (value.title) return value.title;
      return JSON.stringify(value);
    } catch { return ''; }
  }
  if (typeof value === 'boolean') return value ? '是' : '否';
  return String(value);
};

const getFieldValue = (record, fieldName) => {
  const fieldInfo = fieldNameToId.value[fieldName];
  if (!fieldInfo) return '';
  const rawValue = record.fields[fieldInfo.id];
  return formatFieldValue(rawValue, fieldInfo.type);
};

const getFieldName = async (field) => {
  if (field.name && typeof field.name !== 'object') return field.name;
  if (typeof field.getName === 'function') return await field.getName();
  return field.field_id || field.id || '未知字段';
};

// ---------- 核心数据获取 ----------
const fetchAllFieldsDetail = async () => {
  const table = await bitable.base.getActiveTable();
  const fields = await table.getFieldList();
  const detail = [];
  const mapping = {};
  for (const field of fields) {
    const id = field.field_id || field.id;
    const name = await getFieldName(field);
    const type = field.type;
    detail.push({ id, name, type });
    mapping[name] = { id, type };
  }
  return { detail, mapping };
};

const getViewVisibleInfo = async () => {
  const table = await bitable.base.getActiveTable();
  const view = await table.getActiveView();
  const fieldMetaList = await view.getFieldMetaList();
  const visibleFieldIds = await view.getVisibleFieldIdList();
  const visibleMeta = fieldMetaList.filter(meta => visibleFieldIds.includes(meta.id));
  const visibleFieldNames = visibleMeta.map(meta => meta.name);
  const hiddenFieldIds = fieldMetaList.filter(meta => !visibleFieldIds.includes(meta.id)).map(meta => meta.id);
  return { visibleFieldNames, hiddenFieldIds };
};

// ---------- 存储（兼容 bitable.bridge.storage 和 localStorage）----------
let storageAPI = null;

const initStorage = () => {
  if (storageAPI) return storageAPI;
  if (typeof bitable !== 'undefined' && bitable.bridge && bitable.bridge.storage) {
    storageAPI = bitable.bridge.storage;
    console.log('使用 bitable.bridge.storage');
  } else if (typeof localStorage !== 'undefined') {
    storageAPI = localStorage;
    console.warn('bitable.bridge.storage 不可用，使用 localStorage 作为 fallback');
  } else {
    console.error('无可用存储');
    storageAPI = null;
  }
  return storageAPI;
};

const getStorageKey = async (prefix) => {
  const table = await bitable.base.getActiveTable();
  const view = await table.getActiveView();
  return `${prefix}_${table.id}_${view.id}`;
};

const saveExcludedFields = async () => {
  try {
    const storage = initStorage();
    if (!storage) return;
    const key = await getStorageKey('excluded_fields');
    const value = JSON.stringify(excludedFields.value);
    if (storage === localStorage) {
      localStorage.setItem(key, value);
    } else {
      await storage.setItem(key, value);
    }
  } catch (error) { console.error('保存排除字段失败:', error); }
};

const loadExcludedFields = async () => {
  try {
    const storage = initStorage();
    if (!storage) return;
    const key = await getStorageKey('excluded_fields');
    let saved;
    if (storage === localStorage) {
      saved = localStorage.getItem(key);
    } else {
      saved = await storage.getItem(key);
    }
    if (saved) excludedFields.value = JSON.parse(saved);
  } catch (error) { console.error('加载排除字段失败:', error); }
};

const saveTableStyle = async () => {
  try {
    const storage = initStorage();
    if (!storage) return;
    const key = await getStorageKey('table_style');
    const value = JSON.stringify(tableStyle.value);
    if (storage === localStorage) {
      localStorage.setItem(key, value);
    } else {
      await storage.setItem(key, value);
    }
  } catch (error) { console.error('保存表格样式失败:', error); }
};

const loadTableStyle = async () => {
  try {
    const storage = initStorage();
    if (!storage) return;
    const key = await getStorageKey('table_style');
    let saved;
    if (storage === localStorage) {
      saved = localStorage.getItem(key);
    } else {
      saved = await storage.getItem(key);
    }
    if (saved) tableStyle.value = { ...defaultTableStyle, ...JSON.parse(saved) };
  } catch (error) { console.error('加载表格样式失败:', error); }
};

// ---------- 初始化 ----------
const initialize = async () => {
  try {
    const { detail, mapping } = await fetchAllFieldsDetail();
    fieldNameToId.value = mapping;
    const { visibleFieldNames, hiddenFieldIds } = await getViewVisibleInfo();
    const orderedFields = visibleFieldNames.map(name => detail.find(f => f.name === name)).filter(f => f !== undefined);
    allFields.value = orderedFields;
    await loadExcludedFields();
    if (excludedFields.value.length === 0 && hiddenFieldIds.length > 0) {
      const hiddenFieldNames = hiddenFieldIds.map(id => detail.find(f => f.id === id)?.name).filter(name => name !== undefined);
      excludedFields.value = hiddenFieldNames;
    }
    await loadTableStyle();
    await updatePreview();
  } catch (error) {
    console.error('初始化失败:', error);
    ElMessage.error('初始化失败，请检查插件权限');
  }
};

const syncHiddenFields = async () => {
  try {
    const { detail } = await fetchAllFieldsDetail();
    const { hiddenFieldIds } = await getViewVisibleInfo();
    const hiddenFieldNames = hiddenFieldIds.map(id => detail.find(f => f.id === id)?.name).filter(name => name !== undefined);
    excludedFields.value = hiddenFieldNames;
    ElMessage.success(`已同步视图隐藏字段：共 ${hiddenFieldNames.length} 个字段`);
  } catch (error) {
    console.error('同步失败:', error);
    ElMessage.error('同步失败');
  }
};

// ---------- 记录与截图 ----------
const fetchSelectedRecords = async () => {
  try {
    const table = await bitable.base.getActiveTable();
    const view = await table.getActiveView();
    const selectedRecordIds = await view.getSelectedRecordIdList();
    if (!selectedRecordIds.length) return [];
    return await Promise.all(selectedRecordIds.map(rid => table.getRecordById(rid)));
  } catch (error) {
    console.error('获取选中记录失败:', error);
    return [];
  }
};

// ---------- 下载 ----------
const generateAndSave = async () => {
  downloadLoading.value = true;
  try {
    const canvas = await generateCanvas();
    if (!canvas) {
      ElMessage.warning('无法生成图片，请检查是否选中记录且至少有一个字段');
      return;
    }
    const link = document.createElement('a');
    const timestamp = new Date().toISOString().slice(0, 19).replace(/:/g, '-');
    link.download = `表格数据_${timestamp}.png`;
    link.href = canvas.toDataURL();
    link.click();
    ElMessage.success('成功生成表格长图！');
  } catch (error) {
    console.error('生成长图失败:', error);
    ElMessage.error('生成失败，请查看控制台');
  } finally {
    downloadLoading.value = false;
  }
};

const generateCanvas = async () => {
  if (displayedFields.value.length === 0) return null;
  const records = await fetchSelectedRecords();
  if (!records.length) return null;
  selectedRecordsData.value = records;
  await nextTick();
  await new Promise(resolve => setTimeout(resolve, 100));

  const styleElem = document.createElement('style');
  styleElem.textContent = `
    .data-table th, .data-table td {
      border: 1px solid ${tableStyle.value.borderColor};
      padding: 8px 12px;
      text-align: left;
      vertical-align: top;
    }
    .data-table th {
      background-color: ${tableStyle.value.headerBgColor};
      color: ${tableStyle.value.headerTextColor};
      font-weight: 600;
    }
    .data-table tbody tr {
      background-color: ${tableStyle.value.rowBgColor};
      color: ${tableStyle.value.textColor};
      line-height: ${tableStyle.value.lineHeight}px;
    }
    .data-table tbody tr:nth-child(even) {
      background-color: ${tableStyle.value.altRowBgColor};
    }
  `;
  screenshotContainer.value.appendChild(styleElem);
  const canvas = await html2canvas(screenshotContainer.value, { scale: 2, backgroundColor: '#ffffff', logging: false });
  screenshotContainer.value.removeChild(styleElem);
  return canvas;
};

const updatePreview = async () => {
  previewLoading.value = true;
  try {
    const canvas = await generateCanvas();
    previewImageUrl.value = canvas ? canvas.toDataURL() : '';
  } catch (error) {
    console.error('预览生成失败:', error);
    previewImageUrl.value = '';
  } finally {
    previewLoading.value = false;
  }
};

const manualRefreshPreview = () => updatePreview();

let debounceTimer = null;
const debouncedUpdatePreview = () => {
  if (debounceTimer) clearTimeout(debounceTimer);
  debounceTimer = setTimeout(() => updatePreview(), 300);
};

// ---------- 自动刷新：轮询选中记录 + 视图变化 ----------
let lastSelectedIds = [];
let lastViewId = null;
let pollTimer = null;

const getCurrentViewId = async () => {
  try {
    const table = await bitable.base.getActiveTable();
    const view = await table.getActiveView();
    return view.id;
  } catch {
    return null;
  }
};

const getCurrentSelectedIds = async () => {
  try {
    const table = await bitable.base.getActiveTable();
    const view = await table.getActiveView();
    return await view.getSelectedRecordIdList();
  } catch {
    return [];
  }
};

const checkAndRefresh = async () => {
  const currentViewId = await getCurrentViewId();
  if (currentViewId && lastViewId !== currentViewId) {
    console.log('检测到视图变化，重新初始化', lastViewId, '->', currentViewId);
    lastViewId = currentViewId;
    await initialize();
    return;
  }

  const currentIds = await getCurrentSelectedIds();
  let changed = currentIds.length !== lastSelectedIds.length;
  if (!changed) {
    for (let i = 0; i < currentIds.length; i++) {
      if (currentIds[i] !== lastSelectedIds[i]) {
        changed = true;
        break;
      }
    }
  }
  if (changed) {
    console.log('检测到选中记录变化，自动刷新预览');
    lastSelectedIds = currentIds;
    debouncedUpdatePreview();
  }
};

const startPolling = () => {
  if (pollTimer) clearInterval(pollTimer);
  pollTimer = setInterval(() => {
    checkAndRefresh();
  }, 1000);
};

const stopPolling = () => {
  if (pollTimer) {
    clearInterval(pollTimer);
    pollTimer = null;
  }
};

// ---------- 监听与生命周期 ----------
watch(displayedFields, () => debouncedUpdatePreview(), { deep: true });
watch(tableStyle, () => debouncedUpdatePreview(), { deep: true });
watch(excludedFields, () => saveExcludedFields(), { deep: true });

let selectionChangeUnsubscribe = null;

onMounted(async () => {
  await initialize();
  lastSelectedIds = await getCurrentSelectedIds();
  lastViewId = await getCurrentViewId();

  try {
    selectionChangeUnsubscribe = bitable.base.onSelectionChange(() => {
      console.log('事件监听：选中记录变化');
      debouncedUpdatePreview();
      getCurrentSelectedIds().then(ids => { lastSelectedIds = ids; });
    });
  } catch (error) {
    console.error('监听选中变化失败，将仅使用轮询', error);
  }

  startPolling();
});

onUnmounted(() => {
  if (selectionChangeUnsubscribe && typeof selectionChangeUnsubscribe === 'function') {
    selectionChangeUnsubscribe();
  }
  if (debounceTimer) clearTimeout(debounceTimer);
  stopPolling();
});
</script>

<style scoped>
/* 样式保持不变 */
.screenshot-plugin { padding: 20px; background: #f0f2f5; min-height: 100vh; }
.config-card, .preview-card { max-width: 800px; margin: 0 auto 20px; }
.card-header { display: flex; justify-content: space-between; align-items: center; font-size: 18px; font-weight: 600; color: #1f2d3d; }
.preview-tools { display: flex; align-items: center; gap: 12px; flex-wrap: wrap; }
.zoom-controls { display: flex; align-items: center; gap: 8px; flex: 1; min-width: 200px; }
.zoom-label { font-size: 12px; color: #606266; }
.zoom-slider { flex: 1; min-width: 120px; }
.zoom-percent { font-size: 12px; color: #606266; min-width: 40px; text-align: center; }
.field-hint { margin-top: 8px; font-size: 12px; color: #606266; }
.field-hint .el-button { margin-left: 8px; }
.action-buttons { margin-top: 24px; text-align: center; display: flex; gap: 12px; justify-content: center; }
.preview-container { overflow: auto; background: #f5f5f5; border-radius: 8px; padding: 16px; text-align: center; }
.preview-image { display: block; margin: 0 auto; box-shadow: 0 2px 12px rgba(0,0,0,0.1); transition: width 0.2s ease; }
.preview-placeholder { display: flex; flex-direction: column; align-items: center; gap: 12px; color: #909399; font-size: 14px; padding: 40px; }
.screenshot-container { position: fixed; left: -9999px; top: 0; }
</style>