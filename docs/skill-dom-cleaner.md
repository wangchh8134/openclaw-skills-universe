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
