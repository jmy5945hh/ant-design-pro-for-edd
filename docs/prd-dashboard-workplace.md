# PRD - 工作台页面（/dashboard/workplace）

## 1. 总结概要

工作台页面是面向企业内部员工的个性化工作主页，整合了用户信息、项目进展、团队动态、快捷导航及综合能力评分等多个维度的信息。页面以当前登录用户为中心，在页头区域展示个人头像、称谓、职位及统计数字，给予用户归属感和全局视野。主内容区以两栏布局呈现进行中的项目卡片和实时动态流，侧边栏提供快捷操作入口、雷达图评分以及团队成员列表。所有数据通过 React Query 异步加载，充分利用骨架屏保证加载体验流畅。

---

## 2. 业务功能说明

### 2.1 页头个人信息区（PageHeaderContent）

- 展示用户头像、问候语（"早安，{name}，祝你开心每一天！"）
- 展示用户职位与所属部门信息
- 数据加载前显示头像 + 段落骨架屏占位

### 2.2 页头统计数字区（ExtraContent）

右侧展示 3 项个人统计指标：

| 指标 | 说明 |
|------|------|
| 项目数 | 当前用户参与的项目总数 |
| 团队内排名 | 当前用户在团队内的排名（格式：N / 总人数） |
| 项目访问 | 项目页面累计访问量 |

### 2.3 进行中的项目（Project List）

- 以 Grid 卡片形式展示所有进行中项目，每项包含：
  - 项目 logo（小头像）+ 项目名称（可点击跳转）
  - 项目描述文字
  - 负责人/成员姓名链接
  - 最后更新时间（相对时间格式，如"3 天前"）
- 卡片区头部提供「全部项目」快捷链接

### 2.4 团队动态（Activities）

- 展示团队成员最新操作动态的时间流
- 每条动态包含：成员头像、姓名、事件描述（支持模板中嵌入可点击链接）、相对时间
- 动态内容通过 `@{key}` 模板解析，支持动态实体高亮为超链接

### 2.5 快速开始 / 便捷导航

- 固定 6 条可编辑快捷链接，支持通过 `EditableLinkGroup` 组件新增链接
- 导航入口支持自定义 label 与 href，满足不同团队的个性化跳转需求

### 2.6 XX 指数雷达图

- 多维度能力/表现评分雷达图
- 支持多系列对比（通过 `colorField` 区分不同名称数据系列）
- 图例居中显示于雷达图底部

### 2.7 团队成员列表

- 以两列 Grid 形式展示团队成员头像和姓名（截取前 3 字）
- 数据来源复用「进行中项目」接口中的成员字段

---

## 3. UX 设计说明

### 3.1 布局结构

- 使用 `PageContainer` 包裹，统一页面标题区、内容区及 extra 区域
- 主内容区采用 24 栅格双栏布局：
  - 左栏（xl:16）：项目列表 + 动态流
  - 右栏（xl:8）：快捷导航 + 雷达图 + 团队成员
- 所有模块间距统一为 `marginBottom: 24px`

### 3.2 页头设计

- 头像与问候文字水平并排，融入温情化语言提升用户粘性
- 右侧统计数字纵向排列，3 个 Statistic 组件视觉上形成「成就感」区域
- 页头 loading 时整体替换为骨架屏，避免内容闪烁

### 3.3 项目卡片

- `Card.Grid` 实现固定宽度等比例网格布局
- 卡片 meta 区包含图标+标题+描述，底部附成员名与时间戳
- 时间戳采用 `dayjs().fromNow()` 相对时间显示，贴近社交产品体验

### 3.4 动态流

- 使用 antd `List` 组件渲染，large size 保证阅读舒适度
- 动态内容中的实体（用户、项目名等）以链接形式突出显示
- 时间戳作为 description 辅助信息，颜色较淡（`datetime` 样式类）

### 3.5 响应式

- 双栏布局在 lg 及以下断点退化为单栏垂直堆叠（各 Col span=24）
- 项目 Grid 卡片宽度固定，内容区自适应滚动

---

## 4. 技术设计说明

### 4.1 技术栈

| 技术 | 用途 |
|------|------|
| React 18 + TypeScript | 页面主体框架 |
| `@tanstack/react-query` | 数据异步请求与缓存 |
| `@ant-design/plots` `Radar` | 雷达图渲染 |
| `@ant-design/pro-components` `PageContainer` | 页面容器（含页头布局） |
| `antd` Card/List/Avatar/Statistic | UI 基础组件 |
| `dayjs` | 相对时间格式化（`fromNow()`） |
| `antd-style` `createStyles` | 主题 token 驱动样式 |

### 4.2 数据结构

```typescript
// 项目数据
interface ProjectNotice {
  id: string;
  title: string;
  logo: string;
  description: string;
  href: string;
  member: string;
  memberLink: string;
  updatedAt: string;
}

// 动态数据
interface ActivitiesType {
  id: string;
  user: { name: string; avatar: string; link: string };
  template: string;  // 含 @{user}、@{project} 等占位符
  updatedAt: string;
  [key: string]: any; // 动态模板中被引用的实体字段
}

// 当前用户
interface CurrentUser {
  avatar: string;
  name: string;
  userid: string;
  email: string;
  signature: string;
  title: string;
  group: string;
}
```

### 4.3 数据请求

| queryKey | Service 方法 | 接口路径 |
|----------|-------------|----------|
| `project-notice` | `queryProjectNotice()` | `/api/project/notice` |
| `activities` | `queryActivities()` | `/api/activities` |
| `workplace-chart` | `fakeChartData()` | `/api/fake_chart_data` |

所有请求通过 React Query 并发发起，互不阻塞，各自独立控制 loading 状态。

### 4.4 动态模板解析

`renderActivities` 函数使用正则 `/@\{([^{}]*)\}/gi` 解析 `template` 字符串，将 `@{key}` 占位符替换为对应实体的可点击链接，其余文本片段原样渲染。

### 4.5 组件拆分

```
Workplace (index.tsx)
├── PageHeaderContent     — 用户信息页头（带 loading 骨架屏）
├── ExtraContent          — 右侧统计数字区
├── EditableLinkGroup     — 快捷导航链接编辑组（组件）
└── Radar (AntD Plots)    — 雷达图（内联渲染）
```

### 4.6 Mock 数据

Mock 文件：`src/pages/dashboard/workplace/_mock.ts`，提供 `/api/project/notice`、`/api/activities`、`/api/fake_chart_data` 三个接口的模拟响应数据。
