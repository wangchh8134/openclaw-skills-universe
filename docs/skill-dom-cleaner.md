**痛点：** 现代网页一个简单的首页 HTML 可能超过 1MB（几十万个 Token），且充斥着大量对 AI 无效的脚本、样式和深层嵌套。如果直接传给大模型，不仅**成本极高**，还会引入大量的**噪声干扰**，导致定位偏移。

 以下对于`skill-dom-cleaner` 的**整体设计**，旨在实现“极致压缩、语义增强、坐标对齐”。

---

### Skill 产品名称：`skill-dom-cleaner` (极致语义 DOM 压缩器)

#### 1. 设计核心目标 (Core Objectives)
*   **压缩率目标：** 将原始 HTML Token 消耗降低 **85% - 95%**。
*   **语义保留：** 确保所有可交互元素（按钮、输入框、链接）及其上下文描述 100% 保留。
*   **空间感知：** 压缩后的文本必须能够与页面的真实像素坐标（Bounding Box）一一对应。

---

#### 2. 技术架构与处理管线 (Cleaning Pipeline)

我们将清理过程分为四个阶段，采用“**由粗到精**”的过滤策略：

##### 第一阶段：结构性手术 (Structural Pruning)
*   **黑名单剔除：** 暴力移除 `<script>`, `<style>`, `<svg>`, `<path>`, `<iframe>`, `<canvas>`, `<!-- comments -->` 等对视觉理解无意义的标签。
*   **隐藏元素过滤：** 通过 Playwright 检查 `is_visible()` 状态，移除所有 `display: none`、`visibility: hidden` 或透明度为 0 的元素。
*   **无意义属性清理：** 移除所有的内联样式 (`style="..."`)、事件监听器 (`onclick="..."`)、以及复杂的类名 (`class="px-4 py-2 mt-10..."`)。

##### 第二阶段：语义扁平化 (Semantic Flattening)
*   **冗余容器压缩：** 现代网页有大量的 `<div><div><div>文本</div></div></div>`。我们将这些无语义的嵌套直接打平，只保留最内层的文本节点。
*   **角色增强 (ARIA)：** 自动识别 ARIA Role。如果一个 `div` 表现得像一个按钮（`role="button"`），则将其重写为 `<button>`，帮助 LLM 快速识别意图。
*   **表单关联：** 强制将 `<label>` 与对应的 `<input>` 绑定。LLM 必须知道这个输入框是填“手机号”还是“密码”。

##### 第三阶段：交互元素标记 (Interactive Indexing)
*   **Assign ID (AID)：** 给每一个**可交互元素**分配一个唯一的、简短的数字索引（如 `[12]`）。
*   **坐标注入：** 计算每个 AID 元素的中心点坐标 `(x, y)` 和宽高。
*   **输出示例：**
    *   *原始：* `<button class="btn-submit" data-id="99">提交订单</button>`
    *   *压缩后：* `<button id=12 title="提交订单" />` (后台对应坐标 [450, 210])

##### 第四阶段：输出格式化 (Output Generation)
提供两种输出模式：
1.  **Compact Markdown (推荐)：** 适合大多数 LLM，阅读压力最小。
2.  **JSON Tree：** 适合需要进行二次逻辑处理的内部模块。

---

#### 3. 核心功能特性 (Key Features)

| 特性 | 描述 | 价值 |
| :--- | :--- | :--- |
| **视觉优先采样** | 优先保留位于屏幕可视区域（Viewport）内的元素。 | 减少冗余，专注当前操作。 |
| **智能文本截断** | 对于超长的法律条款或隐私协议，仅保留首尾和关键词。 | 防止上下文溢出。 |
| **跨域安全过滤** | 自动识别并脱敏表单中的敏感提示词（如“请输入支付密码”）。 | **L1 安全增强**，保护用户隐私。 |
| **坐标映射表 (Map)** | 维护一个外部状态机，将 ID 映射回真实 DOM 对象。 | 确保 LLM 给出 ID 后，OpenClaw 能瞬间完成点击。 |

---

#### 4. API 接口设计 (Interface Standard)

```typescript
// 输入接口
interface CleanRequest {
  raw_html: string;
  viewport: { width: number; height: number };
  focus_area?: BoundingBox; // 可选：指定重点关注区域
  options: {
    remove_images: boolean;
    compress_text: boolean;
    strip_pii: boolean; // 是否脱敏个人信息
  }
}

// 输出接口
interface CleanResponse {
  cleaned_content: string; // 压缩后的 Markdown 内容
  element_map: Map<number, ElementInfo>; // ID -> {selector, coordinate, rect}
  stats: {
    original_tokens: number;
    cleaned_tokens: number;
    compression_ratio: string;
  }
}
```

---

#### 5. 标杆级演示对比 (Benchmark Demo)

*   **原始页面：** 某电商详情页 (650,000 字符) -> **约 180,000 Tokens**。
*   **`skill-dom-cleaner` 处理后：** (8,000 字符) -> **约 2,200 Tokens**。
*   **压缩效率：** **98.7%**。
*   **效果：** LLM 看到的不再是乱码般的 HTML，而是一个清晰的清单：
    ```markdown
    [1] 搜索输入框 "请输入商品名称"
    [2] 按钮 "搜索"
    [5] 商品标题 "iPhone 15 Pro Max"
    [6] 价格 "￥9,999"
    [12] 按钮 "加入购物车"
    [13] 链接 "查看评价(1000+)"
    ```

---

#### 6. 安全设计 (Security Note)
*   **注入检测：** 在处理过程中，会自动扫描是否存在 `<meta>` 劫持或带有隐藏恶意指令的文本节点。
*   **离线处理：** 该 Skill 设计为在本地执行（不调用外网模型），确保在将数据发给云端 LLM 之前，脱敏工作已经完成。

---

#### 7. 语义压缩算法 **“基于辅助功能树 (Accessibility Tree) 的语义重构算法”**

---

### `skill-dom-cleaner` 核心算法：四步压缩法 (The Claw-Distill Algorithm)

#### 第一步：建立视觉与 DOM 的映射 (Visual-DOM Indexing)
在清理之前，我们先通过 Playwright 对页面进行一次“标记”。
*   **逻辑：** 遍历所有 `is_visible` 的元素，为其添加一个临时的自定义属性 `data-claw-id`。
*   **存储：** 在内存中维护一个 `ElementMap`：
    *   `Key`: Claw-ID (递增整数)
    *   `Value`: `{ selector, bounding_box, role, text_content }`
*   **目的：** 无论后面 HTML 怎么删，我们都能通过 ID 找回它的像素坐标。

#### 第二步：基于辅助功能树的提取 (A-Tree Extraction)
**这是标杆级设计的核心。** 浏览器为残障人士提供的“辅助功能树”本身就是最完美的语义化 HTML。
*   **操作：** 放弃解析原始庞大的 DOM，直接调用浏览器的 Accessibility API。
*   **过滤规则：**
    *   只保留具有 `role` 的节点（如 `button`, `link`, `checkbox`, `heading`, `textbox`）。
    *   **父节点合并：** 如果一个 `div` 下只有一个 `button` 且没有其他文本，直接丢弃 `div`，保留 `button`。
    *   **无交互文本保留：** 只有当文本长度超过阈值或位于标题标签（h1-h6）中时才保留，防止侧边栏碎屑信息干扰。

#### 第三步：属性精简化与语义改写 (Attribute Distillation)
将复杂的 HTML 标签简化为自定义的 **"Claw-Markdown"** 格式，这种格式对 LLM 极其友好。

*   **改写规则示例：**
    *   **Input:** `<input type="text" class="input-v3" placeholder="搜索商品" id="search-123">`
    *   **Distilled:** `[12] input "搜索商品"`
    *   **Button:** `[15] button "立即购买"`
    *   **Link:** `[20] link "帮助中心" -> help.html`
*   **核心逻辑：** 丢弃所有 `class`, `style`, `data-v-xxx` 等干扰字符。只保留 `id` (Claw-ID), `title`, `alt`, `placeholder`, `value`。

#### 第四步：视口裁剪与区域权重 (Viewport Clipping)
根据 Agent 的当前视觉焦点，动态分配 Token 权重。
*   **视口内 (In-Viewport)：** 完整保留语义。
*   **视口外 (Off-screen)：** 
    *   上部：总结为 `--- Above Viewport (Summarized) ---`
    *   下部：仅保留关键导航链接。
*   **逻辑：** 优先保证 LLM 看到的“首屏”是信息密度最高的。

---

### 伪代码逻辑

```python
class ClawDomCleaner:
    def __init__(self, page):
        self.page = page
        self.element_map = {}

    async def distill(self):
        # 1. 获取原始辅助功能树
        snapshot = await self.page.accessibility.snapshot()
        
        # 2. 递归处理节点
        compact_dom = self._process_node(snapshot)
        
        # 3. 结果注入统计信息
        return self._format_output(compact_dom)

    def _process_node(self, node):
        # 核心过滤逻辑
        if not self._is_interesting(node):
            return ""

        # 分配 Claw-ID
        claw_id = self._generate_id(node)
        
        # 属性提取
        role = node.get("role", "text")
        name = node.get("name", "").strip()
        value = node.get("value", "").strip()
        
        # 格式化输出 (Claw-Markdown)
        if role in ["button", "link", "checkbox"]:
            return f"[{claw_id}] {role} '{name}'"
        elif role == "textbox":
            return f"[{claw_id}] input '{name}' (value: '{value}')"
        else:
            # 处理纯文本或容器
            children_text = "".join([self._process_node(c) for c in node.get("children", [])])
            return children_text

    def _is_interesting(self, node):
        # 排除无意义的 generic 容器和空白
        if node.get("role") in ["GenericContainer", "search"] and not node.get("children"):
            return False
        # ... 更多过滤逻辑
        return True
```

---

### **“增量更新 (Incremental Update)”** 技能：

1.  **状态指纹 (DOM Fingerprint)：** 每次压缩后，生成一个摘要指纹。
2.  **变化监测：** 当用户滚动页面时，只更新变动区域的 Claw-ID，而不重置全局索引。
3.  **价值点：** 这样 LLM 能够理解“自己在向下滚动”，且之前的 ID `[12]` 依然有效，极大提升了多轮交互的稳定性。

---

### 核心产品价值

1.  **极致的 Token 节省：** 能够让 Claude 3.5 或 GPT-4o 一次性“看清”比竞争对手多 10 倍的网页内容。
2.  **定位零偏移：** 通过 `ElementMap` 实时维护像素坐标，Agent 不会点到按钮旁边的空白处。
3.  **无视反爬 UI：** 辅助功能树是直接读取渲染引擎的语义，即便网站用了复杂的 CSS 混淆技术，在 Agent 眼里依然清晰如画。
