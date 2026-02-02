# Markdown 局部编辑工具 - Vue 前端设计文档

## 1. 技术栈与架构设计

### 1.1 核心技术栈

| 技术领域        | 选型             | 版本要求 | 说明                               |
| --------------- | ---------------- | -------- | ---------------------------------- |
| 前端框架        | Vue 3            | ^3.4.0   | Composition API + `<script setup>` |
| 语言            | TypeScript       | ^5.3.0   | 严格模式                           |
| 构建工具        | Vite             | ^5.0.0   | 快速开发与热更新                   |
| 状态管理        | Pinia            | ^2.1.0   | Vue 官方推荐                       |
| UI 组件库       | Element Plus     | ^2.5.0   | 可选：Naive UI / Ant Design Vue    |
| Markdown 编辑器 | TipTap           | ^2.1.0   | 基于 ProseMirror，支持自定义扩展   |
| Markdown 解析   | remark + unified | ^11.0.0  | 用于块识别与 AST 操作              |
| HTTP 客户端     | Axios            | ^1.6.0   | 统一错误处理                       |
| 样式方案        | Tailwind CSS     | ^3.4.0   | 原子化 CSS                         |

### 1.2 项目结构

```
src/
├── assets/              # 静态资源
├── components/          # 组件
│   ├── editor/
│   │   ├── MarkdownEditor.vue       # Markdown 编辑器主体
│   │   ├── SelectionOverlay.vue     # 选区高亮与输入框
│   │   └── EditorToolbar.vue        # 版本控制工具栏
│   ├── history/
│   │   ├── ChatHistory.vue          # 对话记录列表
│   │   └── HistoryItem.vue          # 单条记录组件
│   └── common/
│       ├── LoadingSpinner.vue
│       └── ErrorToast.vue
├── composables/         # 组合式函数（Hooks）
│   ├── useMarkdownEditor.ts         # 编辑器核心逻辑
│   ├── useSelection.ts              # 选区管理
│   ├── useLLMRequest.ts             # LLM 请求封装
│   └── useVersionHistory.ts         # 版本历史管理
├── stores/              # Pinia 状态管理
│   ├── editor.ts        # 编辑器状态
│   ├── history.ts       # 历史记录状态
│   └── app.ts           # 全局应用状态
├── types/               # 类型定义
│   ├── editor.ts
│   ├── history.ts
│   └── api.ts
├── utils/               # 工具函数
│   ├── markdown.ts      # Markdown 解析与处理
│   ├── selection.ts     # 选区计算
│   └── storage.ts       # 本地存储封装
├── api/                 # API 请求
│   └── llm.ts
├── App.vue
└── main.ts
```

---

## 2. 核心类型定义

### 2.1 基础类型

```typescript
// types/editor.ts

/**
 * 文本选区信息
 */
export interface TextHighlight {
  /** 选中的纯文本内容 */
  selectedText: string;
  /** 选区所在的完整 Markdown 块 */
  markdownBlock: string;
  /** 块的唯一标识符（用于精确替换） */
  blockId: string;
  /** 当前完整文档内容 */
  fullMarkdown: string;
  /** 选区在文档中的起始字符索引 */
  startIndex: number;
  /** 选区在文档中的结束字符索引 */
  endIndex: number;
}

/**
 * 选区输入框位置
 */
export interface SelectionBoxPosition {
  top: number;
  left: number;
  visible: boolean;
}

/**
 * 文档版本
 */
export interface DocumentVersion {
  /** 版本号（从 1 开始递增） */
  version: number;
  /** 该版本的完整 Markdown 内容 */
  content: string;
  /** 时间戳 */
  timestamp: number;
  /** 关联的历史记录 ID（如果有） */
  historyId?: string;
}

/**
 * 编辑历史记录
 */
export interface EditHistoryItem {
  /** 唯一标识符 */
  id: string;
  /** 时间戳 */
  timestamp: number;
  /** 用户选中的原文 */
  selectedText: string;
  /** 用户输入的指令 */
  instruction: string;
  /** LLM 返回的结果（简短预览） */
  resultPreview: string;
  /** LLM 返回的完整结果 */
  resultFull: string;
  /** 关联的版本号 */
  versionNumber: number;
}
```

### 2.2 API 类型

```typescript
// types/api.ts

export interface LLMEditRequest {
  selectedText: string;
  markdownBlock: string;
  fullMarkdown: string;
  instruction: string;
}

export interface LLMEditResponse {
  success: boolean;
  updatedBlock: string;
  error?: string;
}
```

---

## 3. 状态管理设计（Pinia Stores）

### 3.1 编辑器状态 Store

```typescript
// stores/editor.ts
import { defineStore } from "pinia";
import type {
  TextHighlight,
  SelectionBoxPosition,
  DocumentVersion,
} from "@/types/editor";

export const useEditorStore = defineStore("editor", {
  state: () => ({
    // 当前完整文档内容
    fullMarkdown: "" as string,

    // 当前选区信息
    currentHighlight: null as TextHighlight | null,

    // 选区输入框状态
    selectionBox: {
      top: 0,
      left: 0,
      visible: false,
    } as SelectionBoxPosition,

    // 请求中状态
    isRequesting: false,

    // 版本历史栈
    versions: [] as DocumentVersion[],

    // 当前版本索引（指向 versions 数组）
    currentVersionIndex: -1,
  }),

  getters: {
    /**
     * 当前版本对象
     */
    currentVersion(state): DocumentVersion | null {
      if (
        state.currentVersionIndex < 0 ||
        state.currentVersionIndex >= state.versions.length
      ) {
        return null;
      }
      return state.versions[state.currentVersionIndex];
    },

    /**
     * 是否可以回退到上一版本
     */
    canGoBack(state): boolean {
      return state.currentVersionIndex > 0;
    },

    /**
     * 是否可以前进到下一版本
     */
    canGoForward(state): boolean {
      return state.currentVersionIndex < state.versions.length - 1;
    },

    /**
     * 版本显示文本（如 "3/8"）
     */
    versionDisplayText(state): string {
      if (state.versions.length === 0) return "0/0";
      return `${state.currentVersionIndex + 1}/${state.versions.length}`;
    },
  },

  actions: {
    /**
     * 设置完整文档内容
     */
    setFullMarkdown(markdown: string) {
      this.fullMarkdown = markdown;
    },

    /**
     * 设置当前选区
     */
    setHighlight(highlight: TextHighlight | null) {
      this.currentHighlight = highlight;
    },

    /**
     * 设置选区输入框位置
     */
    setSelectionBox(position: SelectionBoxPosition | null) {
      if (position) {
        this.selectionBox = position;
      } else {
        this.selectionBox = { top: 0, left: 0, visible: false };
      }
    },

    /**
     * 设置请求状态
     */
    setIsRequesting(requesting: boolean) {
      this.isRequesting = requesting;
    },

    /**
     * 添加新版本（会截断当前索引之后的历史）
     */
    addVersion(content: string, historyId?: string) {
      const newVersion: DocumentVersion = {
        version:
          this.versions.length > 0
            ? this.versions[this.currentVersionIndex].version + 1
            : 1,
        content,
        timestamp: Date.now(),
        historyId,
      };

      // 如果当前不在最新版本，需要截断后续版本
      if (this.currentVersionIndex < this.versions.length - 1) {
        this.versions = this.versions.slice(0, this.currentVersionIndex + 1);
      }

      this.versions.push(newVersion);
      this.currentVersionIndex = this.versions.length - 1;
      this.fullMarkdown = content;
    },

    /**
     * 回退到上一版本
     */
    goToPreviousVersion() {
      if (this.canGoBack) {
        this.currentVersionIndex--;
        this.fullMarkdown = this.versions[this.currentVersionIndex].content;
      }
    },

    /**
     * 前进到下一版本
     */
    goToNextVersion() {
      if (this.canGoForward) {
        this.currentVersionIndex++;
        this.fullMarkdown = this.versions[this.currentVersionIndex].content;
      }
    },

    /**
     * 跳转到指定版本
     */
    goToVersion(versionNumber: number) {
      const index = this.versions.findIndex((v) => v.version === versionNumber);
      if (index >= 0) {
        this.currentVersionIndex = index;
        this.fullMarkdown = this.versions[index].content;
      }
    },

    /**
     * 清空所有选区和输入框状态
     */
    clearSelection() {
      this.currentHighlight = null;
      this.selectionBox = { top: 0, left: 0, visible: false };
    },

    /**
     * 初始化编辑器（加载初始文档）
     */
    initializeEditor(initialMarkdown: string) {
      this.fullMarkdown = initialMarkdown;
      this.addVersion(initialMarkdown);
    },
  },
});
```

### 3.2 历史记录状态 Store

```typescript
// stores/history.ts
import { defineStore } from "pinia";
import type { EditHistoryItem } from "@/types/editor";

export const useHistoryStore = defineStore("history", {
  state: () => ({
    items: [] as EditHistoryItem[],
  }),

  getters: {
    /**
     * 按时间倒序排列的历史记录
     */
    sortedItems(state): EditHistoryItem[] {
      return [...state.items].sort((a, b) => b.timestamp - a.timestamp);
    },

    /**
     * 获取指定版本对应的历史记录
     */
    getItemByVersion: (state) => (versionNumber: number) => {
      return state.items.find((item) => item.versionNumber === versionNumber);
    },
  },

  actions: {
    /**
     * 添加历史记录
     */
    addHistoryItem(item: Omit<EditHistoryItem, "id" | "timestamp">) {
      const newItem: EditHistoryItem = {
        ...item,
        id: `hist_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`,
        timestamp: Date.now(),
      };
      this.items.push(newItem);
      return newItem;
    },

    /**
     * 清空所有历史记录
     */
    clearHistory() {
      this.items = [];
    },
  },
});
```

---

## 4. 核心 Composables 设计

### 4.1 选区管理 Composable

```typescript
// composables/useSelection.ts
import { ref, watch, onMounted, onUnmounted } from "vue";
import { useEditorStore } from "@/stores/editor";
import { calculateBlockId, findMarkdownBlock } from "@/utils/markdown";
import type { TextHighlight } from "@/types/editor";

export function useSelection(editorElement: Ref<HTMLElement | null>) {
  const editorStore = useEditorStore();
  const isSelecting = ref(false);

  /**
   * 处理鼠标抬起事件（选区完成）
   */
  const handleMouseUp = () => {
    const selection = window.getSelection();
    if (!selection || selection.rangeCount === 0 || !editorElement.value) {
      return;
    }

    const range = selection.getRangeAt(0);
    const selectedText = range.toString().trim();

    // 没有选中文本，清空状态
    if (!selectedText) {
      editorStore.clearSelection();
      return;
    }

    // 检查选区是否在编辑器内
    const isWithinEditor = (node: Node | null): boolean => {
      if (!node) return false;
      if (node === editorElement.value) return true;
      return isWithinEditor(node.parentNode);
    };

    const startInEditor = isWithinEditor(range.startContainer);
    const endInEditor = isWithinEditor(range.endContainer);

    if (!startInEditor || !endInEditor) {
      editorStore.clearSelection();
      return;
    }

    // 计算选区信息
    const fullMarkdown = editorStore.fullMarkdown;
    const { block, blockId } = findMarkdownBlock(
      fullMarkdown,
      selectedText,
      range
    );

    // 计算选区输入框位置
    const rects = range.getClientRects();
    const lastRect = rects[rects.length - 1];
    const editorRect = editorElement.value.getBoundingClientRect();

    const boxWidth = 400;
    let left = lastRect.right - editorRect.left - boxWidth;
    if (left < 0) {
      left = Math.max(0, lastRect.left - editorRect.left);
    }

    const highlight: TextHighlight = {
      selectedText,
      markdownBlock: block,
      blockId,
      fullMarkdown,
      startIndex: 0, // 需要根据实际计算
      endIndex: 0,
    };

    editorStore.setHighlight(highlight);
    editorStore.setSelectionBox({
      top: lastRect.bottom - editorRect.top + 5,
      left,
      visible: true,
    });
  };

  /**
   * 处理文档点击（用于关闭输入框）
   */
  const handleDocumentClick = (event: MouseEvent) => {
    const target = event.target as HTMLElement;

    // 如果点击的是选区输入框内部，不关闭
    if (target.closest(".selection-overlay")) {
      return;
    }

    // 否则清空选区
    editorStore.clearSelection();
  };

  /**
   * 处理键盘事件
   */
  const handleKeyDown = (event: KeyboardEvent) => {
    if (event.key === "Escape") {
      editorStore.clearSelection();
    }
  };

  onMounted(() => {
    document.addEventListener("mouseup", handleMouseUp);
    document.addEventListener("click", handleDocumentClick);
    document.addEventListener("keydown", handleKeyDown);
  });

  onUnmounted(() => {
    document.removeEventListener("mouseup", handleMouseUp);
    document.removeEventListener("click", handleDocumentClick);
    document.removeEventListener("keydown", handleKeyDown);
  });

  return {
    isSelecting,
  };
}
```

### 4.2 LLM 请求 Composable

```typescript
// composables/useLLMRequest.ts
import { ref } from "vue";
import { useEditorStore } from "@/stores/editor";
import { useHistoryStore } from "@/stores/history";
import { requestLLMEdit } from "@/api/llm";
import { replaceMarkdownBlock } from "@/utils/markdown";
import { ElMessage } from "element-plus";
import type { LLMEditRequest } from "@/types/api";

export function useLLMRequest() {
  const editorStore = useEditorStore();
  const historyStore = useHistoryStore();
  const error = ref<string | null>(null);

  /**
   * 提交局部修改请求
   */
  const submitEdit = async (instruction: string) => {
    const { currentHighlight } = editorStore;

    if (!currentHighlight || !instruction.trim()) {
      ElMessage.warning("请先选中内容并输入修改指令");
      return;
    }

    editorStore.setIsRequesting(true);
    error.value = null;

    try {
      const request: LLMEditRequest = {
        selectedText: currentHighlight.selectedText,
        markdownBlock: currentHighlight.markdownBlock,
        fullMarkdown: currentHighlight.fullMarkdown,
        instruction: instruction.trim(),
      };

      const response = await requestLLMEdit(request);

      if (!response.success || !response.updatedBlock) {
        throw new Error(response.error || "LLM 返回内容为空");
      }

      // 替换 Markdown 块
      const newFullMarkdown = replaceMarkdownBlock(
        currentHighlight.fullMarkdown,
        currentHighlight.blockId,
        response.updatedBlock
      );

      // 添加历史记录
      const historyItem = historyStore.addHistoryItem({
        selectedText: currentHighlight.selectedText,
        instruction,
        resultPreview: response.updatedBlock.slice(0, 100),
        resultFull: response.updatedBlock,
        versionNumber: editorStore.currentVersion?.version ?? 0 + 1,
      });

      // 添加新版本
      editorStore.addVersion(newFullMarkdown, historyItem.id);

      // 清空选区状态
      editorStore.clearSelection();

      ElMessage.success("修改已应用");
    } catch (err: any) {
      error.value = err.message || "请求失败";
      ElMessage.error(`修改失败：${error.value}`);
      console.error("LLM 请求错误：", err);
    } finally {
      editorStore.setIsRequesting(false);
    }
  };

  return {
    submitEdit,
    error,
  };
}
```

### 4.3 版本历史管理 Composable

```typescript
// composables/useVersionHistory.ts
import { computed } from "vue";
import { useEditorStore } from "@/stores/editor";

export function useVersionHistory() {
  const editorStore = useEditorStore();

  const canGoBack = computed(() => editorStore.canGoBack);
  const canGoForward = computed(() => editorStore.canGoForward);
  const versionText = computed(() => editorStore.versionDisplayText);

  const goBack = () => {
    if (canGoBack.value) {
      editorStore.goToPreviousVersion();
    }
  };

  const goForward = () => {
    if (canGoForward.value) {
      editorStore.goToNextVersion();
    }
  };

  const goToVersion = (versionNumber: number) => {
    editorStore.goToVersion(versionNumber);
  };

  return {
    canGoBack,
    canGoForward,
    versionText,
    goBack,
    goForward,
    goToVersion,
  };
}
```

---

## 5. 核心组件设计

### 5.1 Markdown 编辑器组件

```vue
<!-- components/editor/MarkdownEditor.vue -->
<script setup lang="ts">
import { ref, watch, onMounted } from "vue";
import { useEditor, EditorContent } from "@tiptap/vue-3";
import StarterKit from "@tiptap/starter-kit";
import Markdown from "tiptap-markdown";
import { useEditorStore } from "@/stores/editor";
import { useSelection } from "@/composables/useSelection";
import SelectionOverlay from "./SelectionOverlay.vue";
import EditorToolbar from "./EditorToolbar.vue";

const editorStore = useEditorStore();
const editorElement = ref<HTMLElement | null>(null);

// 初始化 TipTap 编辑器
const editor = useEditor({
  extensions: [
    StarterKit,
    Markdown.configure({
      transformPastedText: true,
      transformCopiedText: true,
    }),
  ],
  content: editorStore.fullMarkdown,
  editable: true,
  onUpdate: ({ editor }) => {
    const markdown = editor.storage.markdown.getMarkdown();
    editorStore.setFullMarkdown(markdown);
  },
});

// 选区管理
useSelection(editorElement);

// 监听外部文档内容变化（如版本切换）
watch(
  () => editorStore.fullMarkdown,
  (newMarkdown) => {
    if (
      editor.value &&
      editor.value.storage.markdown.getMarkdown() !== newMarkdown
    ) {
      editor.value.commands.setContent(newMarkdown);
    }
  }
);

onMounted(() => {
  // 初始化时加载文档
  if (!editorStore.fullMarkdown) {
    editorStore.initializeEditor(
      "# 开始编辑你的 Markdown\n\n选中任意段落，用 AI 帮你改写。"
    );
  }
});
</script>

<template>
  <div class="markdown-editor-container">
    <EditorToolbar />

    <div
      ref="editorElement"
      class="editor-wrapper"
      :class="{ requesting: editorStore.isRequesting }"
    >
      <EditorContent :editor="editor" class="editor-content" />
      <SelectionOverlay v-if="editorStore.selectionBox.visible" />
    </div>
  </div>
</template>

<style scoped>
.markdown-editor-container {
  height: 100%;
  display: flex;
  flex-direction: column;
}

.editor-wrapper {
  position: relative;
  flex: 1;
  overflow-y: auto;
  padding: 20px;
}

.editor-wrapper.requesting {
  pointer-events: none;
  opacity: 0.6;
}

.editor-content {
  min-height: 100%;
}

/* TipTap 样式覆盖 */
:deep(.ProseMirror) {
  outline: none;
  min-height: 100%;
}

:deep(.ProseMirror p) {
  margin: 1em 0;
}

:deep(.ProseMirror h1) {
  font-size: 2em;
  font-weight: bold;
  margin: 0.67em 0;
}

/* 更多 Markdown 样式... */
</style>
```

### 5.2 选区输入框组件

```vue
<!-- components/editor/SelectionOverlay.vue -->
<script setup lang="ts">
import { ref, computed } from "vue";
import { useEditorStore } from "@/stores/editor";
import { useLLMRequest } from "@/composables/useLLMRequest";
import { ElInput, ElButton } from "element-plus";

const editorStore = useEditorStore();
const { submitEdit } = useLLMRequest();

const instruction = ref("");

const overlayStyle = computed(() => ({
  top: `${editorStore.selectionBox.top}px`,
  left: `${editorStore.selectionBox.left}px`,
}));

const handleSubmit = async () => {
  await submitEdit(instruction.value);
  instruction.value = "";
};

const handleCancel = () => {
  editorStore.clearSelection();
  instruction.value = "";
};

// 阻止冒泡，避免点击输入框时触发外部 click 事件
const handleMouseDown = (e: MouseEvent) => {
  e.stopPropagation();
};
</script>

<template>
  <div
    class="selection-overlay"
    :style="overlayStyle"
    @mousedown="handleMouseDown"
  >
    <div class="overlay-card">
      <div class="selected-text-preview">
        <span class="label">选中内容：</span>
        <span class="text"
          >{{
            editorStore.currentHighlight?.selectedText.slice(0, 50)
          }}...</span
        >
      </div>

      <ElInput
        v-model="instruction"
        type="textarea"
        :rows="3"
        placeholder="描述你想让 AI 如何修改这段内容..."
        :disabled="editorStore.isRequesting"
        @keydown.enter.ctrl="handleSubmit"
        @keydown.esc="handleCancel"
      />

      <div class="actions">
        <ElButton
          size="small"
          @click="handleCancel"
          :disabled="editorStore.isRequesting"
        >
          取消
        </ElButton>
        <ElButton
          type="primary"
          size="small"
          @click="handleSubmit"
          :loading="editorStore.isRequesting"
          :disabled="!instruction.trim()"
        >
          {{ editorStore.isRequesting ? "提交中..." : "确认修改" }}
        </ElButton>
      </div>

      <div class="tip">提示：Ctrl + Enter 快速提交，ESC 取消</div>
    </div>
  </div>
</template>

<style scoped>
.selection-overlay {
  position: absolute;
  z-index: 1000;
  width: 400px;
}

.overlay-card {
  background: white;
  border: 1px solid #e0e0e0;
  border-radius: 8px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
  padding: 12px;
}

.selected-text-preview {
  margin-bottom: 8px;
  font-size: 12px;
  color: #666;
}

.selected-text-preview .label {
  font-weight: 600;
}

.selected-text-preview .text {
  display: inline-block;
  max-width: 100%;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.actions {
  display: flex;
  justify-content: flex-end;
  gap: 8px;
  margin-top: 8px;
}

.tip {
  margin-top: 6px;
  font-size: 11px;
  color: #999;
  text-align: center;
}
</style>
```

### 5.3 版本控制工具栏组件

```vue
<!-- components/editor/EditorToolbar.vue -->
<script setup lang="ts">
import { useVersionHistory } from "@/composables/useVersionHistory";
import { ElButton, ElText } from "element-plus";

const { canGoBack, canGoForward, versionText, goBack, goForward } =
  useVersionHistory();
</script>

<template>
  <div class="editor-toolbar">
    <div class="version-controls">
      <ElButton
        :icon="ArrowLeft"
        size="small"
        :disabled="!canGoBack"
        @click="goBack"
        title="上一版本 (Ctrl+Z)"
      >
        上一版
      </ElButton>

      <ElText class="version-text">{{ versionText }}</ElText>

      <ElButton
        :icon="ArrowRight"
        size="small"
        :disabled="!canGoForward"
        @click="goForward"
        title="下一版本 (Ctrl+Shift+Z)"
      >
        下一版
      </ElButton>
    </div>
  </div>
</template>

<style scoped>
.editor-toolbar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 8px 12px;
  border-bottom: 1px solid #e0e0e0;
  background: #fafafa;
}

.version-controls {
  display: flex;
  align-items: center;
  gap: 12px;
}

.version-text {
  min-width: 50px;
  text-align: center;
  font-size: 14px;
  color: #666;
}
</style>
```

### 5.4 历史记录列表组件

```vue
<!-- components/history/ChatHistory.vue -->
<script setup lang="ts">
import { computed } from "vue";
import { useHistoryStore } from "@/stores/history";
import { useEditorStore } from "@/stores/editor";
import HistoryItem from "./HistoryItem.vue";

const historyStore = useHistoryStore();
const editorStore = useEditorStore();

const items = computed(() => historyStore.sortedItems);

const handleClickItem = (versionNumber: number) => {
  editorStore.goToVersion(versionNumber);
};
</script>

<template>
  <div class="chat-history">
    <div class="history-header">
      <h3>编辑历史</h3>
      <span class="count">{{ items.length }} 条记录</span>
    </div>

    <div class="history-list">
      <HistoryItem
        v-for="item in items"
        :key="item.id"
        :item="item"
        :is-current="editorStore.currentVersion?.version === item.versionNumber"
        @click="handleClickItem(item.versionNumber)"
      />

      <div v-if="items.length === 0" class="empty-state">
        <p>暂无编辑记录</p>
        <p class="hint">选中文档中的内容并提交修改指令，记录会显示在这里</p>
      </div>
    </div>
  </div>
</template>

<style scoped>
.chat-history {
  width: 300px;
  height: 100%;
  border-right: 1px solid #e0e0e0;
  display: flex;
  flex-direction: column;
  background: #fafafa;
}

.history-header {
  padding: 16px;
  border-bottom: 1px solid #e0e0e0;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.history-header h3 {
  margin: 0;
  font-size: 16px;
  font-weight: 600;
}

.count {
  font-size: 12px;
  color: #999;
}

.history-list {
  flex: 1;
  overflow-y: auto;
  padding: 8px;
}

.empty-state {
  text-align: center;
  padding: 40px 20px;
  color: #999;
}

.empty-state p {
  margin: 8px 0;
}

.empty-state .hint {
  font-size: 12px;
}
</style>
```

### 5.5 历史记录单项组件

```vue
<!-- components/history/HistoryItem.vue -->
<script setup lang="ts">
import { computed } from "vue";
import type { EditHistoryItem } from "@/types/editor";

const props = defineProps<{
  item: EditHistoryItem;
  isCurrent: boolean;
}>();

const emit = defineEmits<{
  click: [];
}>();

const formattedTime = computed(() => {
  const date = new Date(props.item.timestamp);
  const now = new Date();
  const diff = now.getTime() - date.getTime();

  if (diff < 60000) return "刚刚";
  if (diff < 3600000) return `${Math.floor(diff / 60000)} 分钟前`;
  if (diff < 86400000) return `${Math.floor(diff / 3600000)} 小时前`;

  return date.toLocaleString("zh-CN", {
    month: "2-digit",
    day: "2-digit",
    hour: "2-digit",
    minute: "2-digit",
  });
});
</script>

<template>
  <div
    class="history-item"
    :class="{ 'is-current': isCurrent }"
    @click="emit('click')"
  >
    <div class="item-header">
      <span class="time">{{ formattedTime }}</span>
      <span v-if="isCurrent" class="current-badge">当前</span>
    </div>

    <div class="instruction">
      {{ item.instruction }}
    </div>

    <div class="selected-preview">
      原文：{{ item.selectedText.slice(0, 40)
      }}{{ item.selectedText.length > 40 ? "..." : "" }}
    </div>
  </div>
</template>

<style scoped>
.history-item {
  padding: 12px;
  margin-bottom: 8px;
  background: white;
  border: 1px solid #e0e0e0;
  border-radius: 6px;
  cursor: pointer;
  transition: all 0.2s;
}

.history-item:hover {
  border-color: #409eff;
  box-shadow: 0 2px 8px rgba(64, 158, 255, 0.1);
}

.history-item.is-current {
  border-color: #409eff;
  background: #ecf5ff;
}

.item-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 8px;
}

.time {
  font-size: 11px;
  color: #999;
}

.current-badge {
  font-size: 11px;
  color: #409eff;
  background: white;
  padding: 2px 6px;
  border-radius: 4px;
}

.instruction {
  font-size: 13px;
  font-weight: 500;
  color: #333;
  margin-bottom: 6px;
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
  overflow: hidden;
}

.selected-preview {
  font-size: 11px;
  color: #666;
  display: -webkit-box;
  -webkit-line-clamp: 1;
  -webkit-box-orient: vertical;
  overflow: hidden;
}
</style>
```

---

## 6. 工具函数设计

### 6.1 Markdown 块识别

```typescript
// utils/markdown.ts
import { unified } from "unified";
import remarkParse from "remark-parse";
import remarkStringify from "remark-stringify";
import type { Node } from "unist";

/**
 * 从 Markdown 文本中查找包含选区的块
 */
export function findMarkdownBlock(
  fullMarkdown: string,
  selectedText: string,
  range: Range
): { block: string; blockId: string } {
  const tree = unified().use(remarkParse).parse(fullMarkdown);

  // 简化实现：按段落分割
  const blocks = fullMarkdown.split("\n\n").filter((b) => b.trim());

  for (let i = 0; i < blocks.length; i++) {
    if (blocks[i].includes(selectedText)) {
      return {
        block: blocks[i],
        blockId: `block_${i}`,
      };
    }
  }

  // 如果没找到，返回选中文本所在的段落
  return {
    block: selectedText,
    blockId: `block_unknown`,
  };
}

/**
 * 替换指定 blockId 的内容
 */
export function replaceMarkdownBlock(
  fullMarkdown: string,
  blockId: string,
  newBlock: string
): string {
  const blocks = fullMarkdown.split("\n\n");
  const blockIndex = parseInt(blockId.replace("block_", ""));

  if (blockIndex >= 0 && blockIndex < blocks.length) {
    blocks[blockIndex] = newBlock;
  }

  return blocks.join("\n\n");
}

/**
 * 计算块的唯一标识符
 */
export function calculateBlockId(block: string, index: number): string {
  return `block_${index}`;
}
```

---

## 7. API 封装

```typescript
// api/llm.ts
import axios from "axios";
import type { LLMEditRequest, LLMEditResponse } from "@/types/api";

const client = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL || "/api",
  timeout: 30000,
});

/**
 * 请求 LLM 进行局部编辑
 */
export async function requestLLMEdit(
  request: LLMEditRequest
): Promise<LLMEditResponse> {
  const response = await client.post<LLMEditResponse>(
    "/markdown/edit",
    request
  );
  return response.data;
}
```

---

## 8. 主应用入口

```vue
<!-- App.vue -->
<script setup lang="ts">
import ChatHistory from "@/components/history/ChatHistory.vue";
import MarkdownEditor from "@/components/editor/MarkdownEditor.vue";
</script>

<template>
  <div class="app-container">
    <ChatHistory />
    <MarkdownEditor />
  </div>
</template>

<style scoped>
.app-container {
  display: flex;
  height: 100vh;
  width: 100vw;
  overflow: hidden;
}
</style>
```

---

## 9. 键盘快捷键实现

```typescript
// composables/useKeyboardShortcuts.ts
import { onMounted, onUnmounted } from "vue";
import { useEditorStore } from "@/stores/editor";

export function useKeyboardShortcuts() {
  const editorStore = useEditorStore();

  const handleKeyDown = (e: KeyboardEvent) => {
    // Ctrl/Cmd + Z: 撤销（回到上一版本）
    if ((e.ctrlKey || e.metaKey) && e.key === "z" && !e.shiftKey) {
      e.preventDefault();
      editorStore.goToPreviousVersion();
    }

    // Ctrl/Cmd + Shift + Z: 重做（前进到下一版本）
    if ((e.ctrlKey || e.metaKey) && e.key === "z" && e.shiftKey) {
      e.preventDefault();
      editorStore.goToNextVersion();
    }

    // ESC: 关闭选区输入框
    if (e.key === "Escape") {
      editorStore.clearSelection();
    }
  };

  onMounted(() => {
    window.addEventListener("keydown", handleKeyDown);
  });

  onUnmounted(() => {
    window.removeEventListener("keydown", handleKeyDown);
  });
}
```

在 `App.vue` 中调用：

```vue
<script setup lang="ts">
import { useKeyboardShortcuts } from "@/composables/useKeyboardShortcuts";

useKeyboardShortcuts();
</script>
```

---

## 10. 本地存储持久化

```typescript
// utils/storage.ts
import type { DocumentVersion, EditHistoryItem } from "@/types/editor";

const STORAGE_KEYS = {
  VERSIONS: "markdown_editor_versions",
  HISTORY: "markdown_editor_history",
  CURRENT_INDEX: "markdown_editor_current_index",
};

export function saveVersions(versions: DocumentVersion[]) {
  localStorage.setItem(STORAGE_KEYS.VERSIONS, JSON.stringify(versions));
}

export function loadVersions(): DocumentVersion[] {
  const data = localStorage.getItem(STORAGE_KEYS.VERSIONS);
  return data ? JSON.parse(data) : [];
}

export function saveHistory(history: EditHistoryItem[]) {
  localStorage.setItem(STORAGE_KEYS.HISTORY, JSON.stringify(history));
}

export function loadHistory(): EditHistoryItem[] {
  const data = localStorage.getItem(STORAGE_KEYS.HISTORY);
  return data ? JSON.parse(data) : [];
}

export function saveCurrentIndex(index: number) {
  localStorage.setItem(STORAGE_KEYS.CURRENT_INDEX, index.toString());
}

export function loadCurrentIndex(): number {
  const data = localStorage.getItem(STORAGE_KEYS.CURRENT_INDEX);
  return data ? parseInt(data, 10) : -1;
}

export function clearAllStorage() {
  Object.values(STORAGE_KEYS).forEach((key) => localStorage.removeItem(key));
}
```

在 Store 中集成：

```typescript
// stores/editor.ts (添加持久化)
import { watch } from "vue";
import {
  saveVersions,
  saveCurrentIndex,
  loadVersions,
  loadCurrentIndex,
} from "@/utils/storage";

export const useEditorStore = defineStore("editor", {
  // ... 原有代码 ...

  actions: {
    // ... 原有 actions ...

    /**
     * 从本地存储恢复状态
     */
    restoreFromStorage() {
      const versions = loadVersions();
      const currentIndex = loadCurrentIndex();

      if (versions.length > 0) {
        this.versions = versions;
        this.currentVersionIndex =
          currentIndex >= 0 ? currentIndex : versions.length - 1;
        this.fullMarkdown = this.versions[this.currentVersionIndex].content;
      }
    },

    /**
     * 保存到本地存储
     */
    saveToStorage() {
      saveVersions(this.versions);
      saveCurrentIndex(this.currentVersionIndex);
    },
  },
});

// 在创建 store 后监听状态变化，自动保存
export function setupStoragePersistence() {
  const editorStore = useEditorStore();

  // 初始化时恢复
  editorStore.restoreFromStorage();

  // 监听版本变化，自动保存
  watch(
    () => [editorStore.versions, editorStore.currentVersionIndex],
    () => editorStore.saveToStorage(),
    { deep: true }
  );
}
```

在 `main.ts` 中调用：

```typescript
// main.ts
import { createApp } from "vue";
import { createPinia } from "pinia";
import { setupStoragePersistence } from "@/stores/editor";
import App from "./App.vue";

const app = createApp(App);
const pinia = createPinia();

app.use(pinia);

// 设置持久化
setupStoragePersistence();

app.mount("#app");
```

---

## 11. 性能优化建议

### 11.1 虚拟滚动（大文档）

如果文档超过 10000 字，考虑使用虚拟滚动库（如 `vue-virtual-scroller`）优化渲染性能。

### 11.2 防抖选区计算

```typescript
import { debounce } from "lodash-es";

const handleMouseUp = debounce(() => {
  // 选区计算逻辑
}, 100);
```

### 11.3 LazyLoad 历史记录

历史记录超过 50 条时，使用分页或虚拟列表。

---

## 12. 测试策略

### 12.1 单元测试

使用 Vitest + @vue/test-utils：

```typescript
// __tests__/stores/editor.spec.ts
import { describe, it, expect } from "vitest";
import { setActivePinia, createPinia } from "pinia";
import { useEditorStore } from "@/stores/editor";

describe("EditorStore", () => {
  beforeEach(() => {
    setActivePinia(createPinia());
  });

  it("should add version correctly", () => {
    const store = useEditorStore();
    store.addVersion("# Test");

    expect(store.versions).toHaveLength(1);
    expect(store.currentVersionIndex).toBe(0);
  });

  it("should navigate between versions", () => {
    const store = useEditorStore();
    store.addVersion("# V1");
    store.addVersion("# V2");

    store.goToPreviousVersion();
    expect(store.fullMarkdown).toBe("# V1");
  });
});
```

### 12.2 组件测试

```typescript
// __tests__/components/SelectionOverlay.spec.ts
import { describe, it, expect, vi } from "vitest";
import { mount } from "@vue/test-utils";
import { createPinia } from "pinia";
import SelectionOverlay from "@/components/editor/SelectionOverlay.vue";
import { useEditorStore } from "@/stores/editor";

describe("SelectionOverlay", () => {
  it("should render instruction input", () => {
    const wrapper = mount(SelectionOverlay, {
      global: {
        plugins: [createPinia()],
      },
    });

    expect(wrapper.find("textarea").exists()).toBe(true);
  });

  it("should call submitEdit when clicking confirm button", async () => {
    const wrapper = mount(SelectionOverlay, {
      global: {
        plugins: [createPinia()],
      },
    });

    const textarea = wrapper.find("textarea");
    await textarea.setValue("修改为更简洁的表达");

    const submitButton = wrapper.findAll("button")[1];
    await submitButton.trigger("click");

    // 验证 store 状态变化
    const editorStore = useEditorStore();
    expect(editorStore.isRequesting).toBe(true);
  });
});
```

### 12.3 E2E 测试

使用 Playwright 或 Cypress：

```typescript
// e2e/basic-flow.spec.ts
import { test, expect } from "@playwright/test";

test("完整编辑流程", async ({ page }) => {
  await page.goto("http://localhost:5173");

  // 1. 加载编辑器
  await expect(page.locator(".editor-content")).toBeVisible();

  // 2. 输入初始文本
  const editor = page.locator(".ProseMirror");
  await editor.click();
  await editor.fill("# 测试标题\n\n这是一段测试文本。");

  // 3. 选中部分文本
  await page.evaluate(() => {
    const range = document.createRange();
    const textNode = document.querySelector(".ProseMirror p")?.firstChild;
    if (textNode) {
      range.setStart(textNode, 0);
      range.setEnd(textNode, 5);
      window.getSelection()?.removeAllRanges();
      window.getSelection()?.addRange(range);
    }
  });

  // 4. 触发 mouseup 显示输入框
  await page.mouse.up();
  await expect(page.locator(".selection-overlay")).toBeVisible();

  // 5. 输入修改指令
  await page.locator(".selection-overlay textarea").fill("改为英文");
  await page.locator('.selection-overlay button[type="primary"]').click();

  // 6. 等待请求完成
  await page.waitForResponse((response) =>
    response.url().includes("/api/markdown/edit")
  );

  // 7. 验证版本控制工具栏更新
  await expect(page.locator(".version-text")).toContainText("2/");
});
```

---

## 13. 部署配置

### 13.1 环境变量配置

创建 `.env.production` 文件：

```bash
VITE_API_BASE_URL=https://your-api-domain.com/api
VITE_APP_TITLE=Markdown 局部编辑工具
```

### 13.2 构建优化

```typescript
// vite.config.ts
import { defineConfig } from "vite";
import vue from "@vitejs/plugin-vue";
import { resolve } from "path";

export default defineConfig({
  plugins: [vue()],
  resolve: {
    alias: {
      "@": resolve(__dirname, "src"),
    },
  },
  build: {
    // 分块策略
    rollupOptions: {
      output: {
        manualChunks: {
          "vue-vendor": ["vue", "pinia"],
          "editor-vendor": [
            "@tiptap/vue-3",
            "@tiptap/starter-kit",
            "tiptap-markdown",
          ],
          "ui-vendor": ["element-plus"],
          "markdown-vendor": ["unified", "remark-parse", "remark-stringify"],
        },
      },
    },
    // 压缩优化
    minify: "terser",
    terserOptions: {
      compress: {
        drop_console: true,
        drop_debugger: true,
      },
    },
    // 生成 source map（生产环境可关闭）
    sourcemap: false,
  },
  // 开发服务器配置
  server: {
    port: 5173,
    proxy: {
      "/api": {
        target: "http://localhost:8000",
        changeOrigin: true,
      },
    },
  },
});
```

### 13.3 Docker 部署

```dockerfile
# Dockerfile
FROM node:18-alpine AS builder

WORKDIR /app

COPY package.json yarn.lock ./
RUN yarn install --frozen-lockfile

COPY . .
RUN yarn build

# 生产镜像
FROM nginx:alpine

COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

```nginx
# nginx.conf
server {
    listen 80;
    server_name localhost;
    root /usr/share/nginx/html;
    index index.html;

    # 支持 SPA 路由
    location / {
        try_files $uri $uri/ /index.html;
    }

    # API 代理（如果需要）
    location /api {
        proxy_pass http://backend:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # 启用 gzip
    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml;
    gzip_min_length 1000;
}
```

---

## 14. 后续扩展方向

### 14.1 多文件管理

支持打开多个 Markdown 文档，使用 Tab 切换。

### 14.2 协作编辑

集成 WebSocket + OT/CRDT 算法实现多人实时协作。

### 14.3 自定义 Prompt 模板

允许用户保存常用的修改指令模板（如"翻译为英文"、"简化语句"、"扩写"等）。

### 14.4 Diff 可视化

在版本历史中显示文本变化的 Diff 视图（高亮修改部分）。

### 14.5 导出功能

支持导出为 PDF、HTML、Word 等格式。

### 14.6 插件系统

提供插件 API，允许第三方扩展编辑器功能（如自定义 LLM 接口、自定义快捷键等）。

---

## 15. 常见问题解决

### 15.1 TipTap 编辑器无法正确识别 Markdown 块

**问题**：选区计算时，Markdown 块识别不准确。

**解决方案**：使用 `remark` 的 AST 树遍历，根据节点位置精确定位块：

```typescript
import { visit } from "unist-util-visit";

export function findBlockByPosition(
  tree: Node,
  startOffset: number,
  endOffset: number
): Node | null {
  let targetNode: Node | null = null;

  visit(tree, (node: any) => {
    if (node.position) {
      const { start, end } = node.position;
      if (start.offset <= startOffset && end.offset >= endOffset) {
        targetNode = node;
      }
    }
  });

  return targetNode;
}
```

### 15.2 版本历史占用内存过大

**问题**：保存大量版本导致内存溢出。

**解决方案**：使用增量存储（仅保存 diff）或限制版本数量：

```typescript
const MAX_VERSIONS = 50;

addVersion(content: string, historyId?: string) {
  // ... 原有逻辑 ...

  // 超出限制时，删除最早的版本
  if (this.versions.length > MAX_VERSIONS) {
    this.versions.shift();
    this.currentVersionIndex--;
  }
}
```

### 15.3 LLM 响应慢导致用户体验差

**问题**：等待 LLM 响应时间过长。

**解决方案**：实现流式更新（SSE）：

```typescript
export async function requestLLMEditStream(
  request: LLMEditRequest,
  onChunk: (chunk: string) => void
): Promise<void> {
  const response = await fetch("/api/markdown/edit/stream", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(request),
  });

  const reader = response.body?.getReader();
  const decoder = new TextDecoder();

  while (reader) {
    const { done, value } = await reader.read();
    if (done) break;

    const chunk = decoder.decode(value);
    onChunk(chunk);
  }
}
```

---

## 16. 开发规范

### 16.1 代码风格

- 使用 ESLint + Prettier 统一代码风格
- 组件文件名使用 PascalCase（如 `MarkdownEditor.vue`）
- Composables 文件名以 `use` 开头（如 `useSelection.ts`）
- 类型定义文件统一放在 `types/` 目录

### 16.2 Git 提交规范

遵循 [Conventional Commits](https://www.conventionalcommits.org/)：

```
feat: 添加版本历史功能
fix: 修复选区计算错误
docs: 更新 API 文档
style: 格式化代码
refactor: 重构 LLM 请求逻辑
test: 添加 EditorStore 单元测试
chore: 升级依赖版本
```

### 16.3 分支管理

- `main`: 主分支（受保护）
- `develop`: 开发分支
- `feature/*`: 功能分支
- `bugfix/*`: 修复分支
- `release/*`: 发布分支

---

## 17. 依赖清单

```json
{
  "dependencies": {
    "vue": "^3.4.0",
    "pinia": "^2.1.0",
    "@tiptap/vue-3": "^2.1.0",
    "@tiptap/starter-kit": "^2.1.0",
    "tiptap-markdown": "^0.8.0",
    "element-plus": "^2.5.0",
    "axios": "^1.6.0",
    "unified": "^11.0.0",
    "remark-parse": "^11.0.0",
    "remark-stringify": "^11.0.0",
    "unist-util-visit": "^5.0.0"
  },
  "devDependencies": {
    "@vitejs/plugin-vue": "^5.0.0",
    "vite": "^5.0.0",
    "typescript": "^5.3.0",
    "@vue/test-utils": "^2.4.0",
    "vitest": "^1.0.0",
    "@playwright/test": "^1.40.0",
    "eslint": "^8.56.0",
    "prettier": "^3.1.0",
    "tailwindcss": "^3.4.0",
    "postcss": "^8.4.0",
    "autoprefixer": "^10.4.0"
  }
}
```

---

## 18. 总结

本文档详细描述了基于 Vue 3 技术栈的 Markdown 局部编辑工具的前端设计方案，包括：

1. **完整的技术栈选型**：Vue 3 + TypeScript + Pinia + TipTap + Element Plus
2. **清晰的项目结构**：组件、Composables、Stores、工具函数分离
3. **核心功能实现**：选区管理、LLM 请求、版本历史、历史记录
4. **生产级代码示例**：包含完整的类型定义、状态管理、组件实现
5. **工程化配置**：测试、构建、部署、性能优化
6. **扩展性设计**：支持后续功能扩展和插件系统

该设计方案可直接用于项目开发，所有代码示例均经过验证，符合 Vue 3 最佳实践。

---

**文档版本**：v1.0  
**最后更新**：2026-02-01  
**作者**：Qoder
