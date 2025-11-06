# 金融演示文稿设计规范

## 设计风格
**Deck Style**: Light / Bank Report  
整体风格清爽专业，强调可读性与数据呈现。

## 页面尺寸与布局
- **尺寸**: 1920x1080px (16:9 标准演示文稿)
- **Padding**: 48px - 使用内联样式 `style={{ paddingLeft: '48px', paddingRight: '48px', paddingTop: '32px', paddingBottom: '32px' }}`
- **背景色**: #F7FAFF (subtle light)
- **用途**: 通过 Figma Copy Design 功能直接复制到 Figma Slides 中，无需手动调整即可完美适配

## 配色方案
- **Primary**: #366FF9
- **Secondary**: #69ACF9
- **Accent**: #F5A623
- **Text Dark**: #1F2937
- **Text Medium**: #374151
- **Text Light**: #64748B
- **Border**: #E5EAF2

⚠️ **颜色使用限制**：图表系列仅使用 Primary/Secondary/Accent，避免过多颜色

## 文字规范（1920x1080大屏适配）
- **主标题 (Title/h1)**: 64px / semibold
- **副标题 (H2)**: 40px / semibold
- **图表标签**: 22px / medium
- **图表图例**: 22px / regular
- **列表文字**: 32px / regular
- **正文 (Body)**: 22px / regular
- **说明文字 (Caption)**: 13px / regular (仅用于次要信息)

⚠️ **重要**: 必须使用内联样式显式设置字号和字重，覆盖组件默认样式

## 卡片样式
- **背景**: 白色 (#FFFFFF)
- **圆角**: 16px (rounded-2xl)
- **边框**: 1px solid #E5EAF2
- **阴影**: `boxShadow: '0 8px 24px rgba(0, 0, 0, 0.1)'` (y=8, blur=24, opacity=10%)
- **内边距**: p-8 或 p-10
- **拉伸**: 使用 `flex-1` 和 `minWidth: 0` 确保卡片在Figma中可自适应拉伸

## 图表规范
### 通用设置
- **背景**: 浅色 (light background)
- **网格线**: #E7EDF5
- **坐标轴文字**: #374151
- **系列颜色**: 仅使用 Primary (#366FF9) / Secondary (#69ACF9) / Accent (#F5A623)
- **ResponsiveContainer**: width="100%" height="100%"

### 饼图特定
- **外半径 (outerRadius)**: 200px (适配1920x1080大屏)
- **标签字号**: 22px / medium
- **图例高度**: 60-70px
- **图例字号**: 22px
- **图标尺寸**: 14px
- **Tooltip**: fontSize 20px, padding 14px

### 其他图表
- **柱状图/折线图**: 使用recharts组件，严格遵循配色方案
- **gridlines**: stroke="#E7EDF5"
- **axis labels**: fill="#374151", fontSize="14px"

## 图标规范
- **风格**: outline/duotone
- **线条**: single-line weight
- **颜色**: 仅使用 Primary (#366FF9)
- **来源**: lucide-react

## 可编辑文本规则
✅ **必须使用**: contentEditable + suppressContentEditableWarning
```tsx
<h2 
  style={{ fontSize: '40px', fontWeight: 600 }}
  contentEditable
  suppressContentEditableWarning
>
  标题文本
</h2>
```

❌ **禁止使用**: `<input>` 或 `<textarea>` 元素

**原因**: contentEditable确保文字在Figma Copy Design中正确显示和复制。所有文本和图表必须可编辑。

## Figma Copy Design 兼容性规则
⚠️ **关键限制**: 避免 contentEditable 内部使用局部样式标签

❌ **禁止**: 在 contentEditable 元素内部使用 `<span>` 标签设置局部样式
```tsx
// ❌ 错误示例 - 会导致Figma中字体加深和漂移
<p contentEditable suppressContentEditableWarning>
  用<span style={{ fontWeight: 600 }}>公开指标</span>推导资产收益
</p>
```

✅ **推荐**: 保持 contentEditable 内部文字样式统一
```tsx
// ✅ 正确示例 - Figma完美复制
<p contentEditable suppressContentEditableWarning>
  用公开指标推导资产收益
</p>
```

✅ **替代方案**: 如需强调，拆分为独立的 contentEditable 元素
```tsx
// ✅ 正确示例 - 使用多个独立元素
<div style={{ display: 'flex', gap: '4px' }}>
  <p contentEditable suppressContentEditableWarning>用</p>
  <p style={{ fontWeight: 600 }} contentEditable suppressContentEditableWarning>公开指标</p>
  <p contentEditable suppressContentEditableWarning>推导资产收益</p>
</div>
```

**根本原因**: 
- `<span>` 嵌套在 contentEditable 中会干扰浏览器的编辑行为
- Figma Copy Design 时会错误解析嵌套的样式标签
- 导致字体渲染异常（加深、漂移、重叠等问题）

**适用范围**:
- ❌ 不要在 contentEditable 内使用: `<span>`, `<b>`, `<strong>`, `<i>`, `<em>` 等
- ✅ 只在 contentEditable 元素的容器级别设置统一样式
- ✅ 使用内联 `style` 属性在 contentEditable 元素本身设置样式

## 布局原则
### ✅ Do
- **左对齐**: 所有内容默认左对齐
- **充足留白**: 主要区块间距 gap-12，标题与内容 mb-8 到 mb-10
- **列表间距**: space-y-6 到 space-y-7
- **清晰层级**: 使用文字大小和间距建立视觉层级

### ❌ Don't
- 深色背景 (dark backgrounds)
- 渐变效果 (gradients)
- 厚重阴影 (heavy shadows)
- 过多颜色

## 样式覆盖提醒
⚠️ **组件默认样式**: Shadcn/ui等基础组件可能有内置的gap/typography默认值，**必须**在生成的React代码中显式设置所有样式参数以覆盖默认值。
