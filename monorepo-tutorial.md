# 《从零理解并实战 Monorepo 架构》

> 适合人群：前端初中级开发者（React/Vue + Node 基础，工程化经验不系统）
>
> 目标：学完后，你可以**独立设计并维护一个中大型 Monorepo 项目**。

---

## 第一章：为什么我们需要 Monorepo

### 本章学习目标
- 理解团队在多人协作时，为什么会被“仓库拆太散”拖慢速度。
- 看懂 Monorepo 与 Multi-repo 的核心差异。
- 知道 Monorepo 在中大型项目中的价值边界。

### 核心概念解释（通俗版）
想象你开了一家连锁餐饮：
- **Multi-repo**：每家店（前端项目）自己采购、自己定菜单、自己管理库存（依赖），沟通成本高，标准难统一。
- **Monorepo**：所有门店由一个中央仓库统一采购和规则管理。每家店仍可独立运营，但标准一致、物料共享快。

#### Monorepo vs Multi-repo 对比

| 维度 | Multi-repo | Monorepo |
|---|---|---|
| 代码组织 | 一个项目一个仓库 | 多个项目在同一仓库 |
| 依赖升级 | 各仓库分别升级，容易版本漂移 | 集中升级，版本一致性更高 |
| 跨项目改动 | 需跨多个仓库提PR | 一个仓库内可原子提交 |
| CI 复杂度 | 多仓库流水线分散 | 可统一设计（按变更范围执行） |
| 权限隔离 | 天然更强 | 需通过目录规范和流程控制 |
| 适用场景 | 团队小、项目独立性强 | 中大型团队、共享代码多 |

### 为什么中大型项目需要 Monorepo
当你有多个前端应用（官网、管理台、H5）、多个基础包（UI、工具、配置）时，常见痛点是：
1. **重复造轮子**：每个项目都写一套工具函数。
2. **版本地狱**：A 项目用 ui@1.2，B 用 ui@1.6，Bug 不一致。
3. **联动发布困难**：改了 utils，要在多个仓库重复发版、提 MR。
4. **新人上手慢**：需要 clone 多个仓库才看全全貌。

Monorepo 的价值在于：
- 统一依赖与规范；
- 跨包改动可以一次完成；
- 基础设施可共享（lint、tsconfig、构建脚本、CI 模板）；
- 更适合“平台化”演进。

### 实战示例（思维演示）
假设你现在有三个仓库：
- `admin-web`
- `h5-web`
- `shared-utils`

某天你修改了手机号校验逻辑：
- Multi-repo：改 `shared-utils` -> 发版 -> admin 升级 -> h5 升级 -> 分别验证。
- Monorepo：同一个提交里改 `packages/utils`、`apps/admin`、`apps/h5`，一次 CI 验证。

### 常见错误
- 误以为 Monorepo = “一个项目拆很多目录”。
- 不评估团队规模和协作方式，盲目跟风。
- 没有配套规范（代码所有权、变更流程、发布策略），导致“超级大杂烩仓库”。

### 小结
Monorepo 不是银弹，但对“多应用 + 多共享包 + 频繁联动”场景非常高效。关键不在“放一起”，而在“统一治理 + 模块边界清晰”。

---

## 第二章：Monorepo 基础概念

### 本章学习目标
- 理解 workspace、包边界、依赖提升、软链接等核心术语。
- 掌握 pnpm workspace 的工作原理。
- 能解释“子包互相引用”到底发生了什么。

### 核心概念解释（通俗版）
把 Monorepo 想象成“一个大园区”：
- `apps/` 是对外营业的店铺（可运行应用）。
- `packages/` 是园区公共服务中心（UI、工具、配置）。
- workspace 像园区总调度：负责登记每个包、处理包之间的交通（依赖关系）。

### pnpm workspace 原理（重点）
`pnpm` 的核心是“**内容寻址存储 + 硬链接/符号链接**”：
1. 依赖包先下载到全局 store（避免重复下载）。
2. 每个项目的 `node_modules` 通过链接指向 store 中的真实内容。
3. workspace 内部包（比如 `@book/utils`）优先按本地包链接，而不是去 npm 下载。

好处：
- 安装更快、占用更省。
- 依赖关系更严格，能更早暴露“幽灵依赖”（没声明却偷用）。

### 包依赖管理机制
在 Monorepo 中通常分三类依赖：
- **外部依赖**：如 `react`、`axios`。
- **内部依赖**：如 `@book/ui` 依赖 `@book/utils`。
- **开发依赖**：构建、测试、lint 工具。

推荐：
- 用 `workspace:*` 明确内部依赖。
- 把公共工具链依赖放根目录统一版本。

### 子包之间如何互相引用（示例）
```json
{
  "name": "@book/ui",
  "version": "0.1.0",
  "dependencies": {
    "@book/utils": "workspace:*"
  }
}
```

`@book/ui` 中这样写：
```ts
import { formatDate } from '@book/utils'
```

pnpm 会把它解析到本地 `packages/utils`，不是远程 npm。

### 实战示例（最小化配置）
根目录 `pnpm-workspace.yaml`：
```yaml
packages:
  - 'apps/*'
  - 'packages/*'
```

### 常见错误
- 忘记把目录加到 `pnpm-workspace.yaml`，导致包“不在工作区”。
- 内部包用普通版本号（如 `^1.0.0`）而不是 `workspace:*`，导致误装远程版本。
- 子包 `name` 写错，导入路径与包名不一致。

### 小结
你可以把 pnpm workspace 理解成“包关系路由器”：负责识别谁是谁、谁依赖谁、依赖从哪里来。

---

## 第三章：实战搭建第一个 Monorepo

### 本章学习目标
- 从 0 初始化一个可运行 Monorepo。
- 完成 `apps/` 与 `packages/` 基础结构。
- 跑通“应用引用内部包”的闭环。

### 核心概念解释（通俗版）
先搭“地基”再装“房间”：
- 地基：根目录统一配置（workspace、脚本、TS 基座）。
- 房间：apps 和 packages 各自职责清晰。

### 实战示例（完整代码）

#### 1）目录结构
```txt
book-monorepo/
  apps/
    web/
  packages/
    ui/
    utils/
    config/
  package.json
  pnpm-workspace.yaml
  tsconfig.base.json
```

#### 2）根目录 `package.json`
```json
{
  "name": "book-monorepo",
  "private": true,
  "packageManager": "pnpm@9.0.0",
  "scripts": {
    "dev:web": "pnpm --filter @book/web dev",
    "build": "pnpm -r build",
    "typecheck": "pnpm -r typecheck"
  },
  "devDependencies": {
    "typescript": "^5.4.0"
  }
}
```

#### 3）`pnpm-workspace.yaml`
```yaml
packages:
  - 'apps/*'
  - 'packages/*'
```

#### 4）`packages/utils`
`packages/utils/package.json`
```json
{
  "name": "@book/utils",
  "version": "0.0.1",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "scripts": {
    "build": "tsc -p tsconfig.json",
    "typecheck": "tsc -p tsconfig.json --noEmit"
  }
}
```

`packages/utils/src/index.ts`
```ts
export function formatDate(input: Date): string {
  const y = input.getFullYear()
  const m = String(input.getMonth() + 1).padStart(2, '0')
  const d = String(input.getDate()).padStart(2, '0')
  return `${y}-${m}-${d}`
}
```

#### 5）`packages/ui`
`packages/ui/package.json`
```json
{
  "name": "@book/ui",
  "version": "0.0.1",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "dependencies": {
    "@book/utils": "workspace:*"
  },
  "scripts": {
    "build": "tsc -p tsconfig.json",
    "typecheck": "tsc -p tsconfig.json --noEmit"
  }
}
```

`packages/ui/src/index.ts`
```ts
import { formatDate } from '@book/utils'

export function getTodayLabel() {
  return `今天是 ${formatDate(new Date())}`
}
```

#### 6）`apps/web`
`apps/web/package.json`
```json
{
  "name": "@book/web",
  "version": "0.0.1",
  "private": true,
  "dependencies": {
    "@book/ui": "workspace:*"
  },
  "scripts": {
    "dev": "node src/main.js",
    "build": "echo build web",
    "typecheck": "echo skip"
  }
}
```

`apps/web/src/main.js`
```js
const { getTodayLabel } = require('@book/ui')
console.log(getTodayLabel())
```

#### 7）安装与运行
```bash
pnpm install
pnpm build
pnpm dev:web
```

### 常见错误
- 直接在子包里 `npm install`，破坏 workspace 一致性。
- 没有构建 `@book/ui` 就在 `apps/web` 里运行（dist 不存在）。
- ESM/CJS 混用导致导入失败（后面第 5 章会讲解）。

### 小结
到这一步，你已经拥有了一个可运行的 Monorepo 雏形：应用 -> 内部 UI 包 -> 工具包，链路清晰。

---

## 第四章：模块拆分与依赖设计

### 本章学习目标
- 学会按职责拆分 `apps` 与 `packages`。
- 理解“依赖方向”与“边界约束”。
- 能设计可扩展的包结构。

### 核心概念解释（通俗版）
把系统当成城市交通：
- 主干道（核心包）尽量稳定。
- 支路（业务应用）变化频繁。
- 交通规则（依赖方向）必须明确，不然全城堵车（循环依赖）。

### 推荐结构
```txt
apps/
  admin/
  web/
packages/
  ui/
  utils/
  config/
  api-client/
```

### 依赖设计原则
1. `apps/*` 可以依赖 `packages/*`。
2. `packages/ui` 可以依赖 `packages/utils`，反过来不行（避免倒挂）。
3. `packages/config` 只放配置，不放业务逻辑。
4. 避免横向乱依赖，保持“层级单向流动”。

### 实战示例（依赖边界）
`packages/config/package.json`
```json
{
  "name": "@book/config",
  "version": "0.0.1",
  "exports": {
    "./eslint": "./eslint/index.js",
    "./tsconfig": "./tsconfig/base.json"
  }
}
```

`apps/admin/package.json`
```json
{
  "name": "@book/admin",
  "private": true,
  "devDependencies": {
    "@book/config": "workspace:*"
  }
}
```

### 常见错误
- 把所有代码都放 `utils`，最后变成“垃圾抽屉包”。
- 子包职责不明确（UI 包里出现接口请求逻辑）。
- 循环依赖（A -> B -> A）导致构建和运行异常。

### 小结
拆分不是越碎越好，而是“按变化频率与职责边界拆分”，让协作和维护成本下降。

---

## 第五章：工程化配置（TS / 构建）

### 本章学习目标
- 掌握 Monorepo 下 TypeScript 配置层级。
- 了解构建流程设计（并行、缓存、按需构建）。
- 解决常见的模块系统与类型声明问题。

### 核心概念解释（通俗版）
把 TS 配置理解为“校规”：
- 总校规（根 `tsconfig.base.json`）统一底线。
- 分校规（子包 `tsconfig.json`）按需微调。

### TypeScript 配置方式（推荐）
#### 根配置 `tsconfig.base.json`
```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "CommonJS",
    "declaration": true,
    "strict": true,
    "skipLibCheck": true,
    "esModuleInterop": true,
    "baseUrl": ".",
    "paths": {
      "@book/utils": ["packages/utils/src"],
      "@book/ui": ["packages/ui/src"]
    }
  }
}
```

#### 子包配置 `packages/ui/tsconfig.json`
```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "outDir": "dist",
    "rootDir": "src"
  },
  "include": ["src"]
}
```

### 构建流程设计
初期可用 `pnpm -r build`，中后期建议引入任务编排工具（如 Turborepo/Nx）实现：
- 按依赖拓扑排序构建。
- 并行执行无关任务。
- 基于缓存跳过未变更任务。

#### 基础构建命令
```bash
pnpm -r --filter ./packages/* build
pnpm -r --filter ./apps/* build
```

### 常见错误
- 所有包复制一份 `tsconfig`，长期漂移。
- `paths` 只在 TS 生效，运行时没对应 alias 解析。
- build 顺序错误：先构建 app，后构建依赖包。

### 小结
工程化的关键是“统一 + 分层 + 自动化”，不是文件越多越专业。

---

## 第六章：真实项目落地案例

### 本章学习目标
- 通过一个完整业务场景串联前面知识。
- 能把 Monorepo 用于真实团队协作。
- 明确从开发到发布的全流程。

### 核心概念解释（通俗版）
案例：电商中台
- `apps/admin`：运营后台（React）
- `apps/h5`：移动端活动页（Vue）
- `packages/ui`：统一组件
- `packages/utils`：公共方法
- `packages/config`：eslint/tsconfig/shared scripts

### 实战示例（联动开发）
需求：后台和 H5 都要新增“人民币格式化展示”。

`packages/utils/src/money.ts`
```ts
export function formatCNY(amount: number): string {
  return `¥${amount.toFixed(2)}`
}
```

`packages/utils/src/index.ts`
```ts
export * from './money'
```

`apps/admin/src/demo.ts`
```ts
import { formatCNY } from '@book/utils'
console.log(formatCNY(199)) // ¥199.00
```

`apps/h5/src/demo.ts`
```ts
import { formatCNY } from '@book/utils'
console.log(formatCNY(88.8)) // ¥88.80
```

### 发布策略
常见两种：
1. **锁步发布（Fixed/Locked）**：所有包同版本。
   - 优点：简单一致。
   - 缺点：小改动也全体升级。
2. **独立发布（Independent）**：每个包独立版本。
   - 优点：灵活。
   - 缺点：版本治理复杂。

建议：
- 团队早期先锁步，流程跑顺后再独立发布。
- 使用 Changesets 管理版本与 changelog。

### 常见错误
- 没有约定“什么变更触发什么版本号”。
- 忽略变更日志，线上问题难回溯。
- 发包前未做“受影响范围测试”。

### 小结
Monorepo 落地不只是目录结构，更是“开发协同 + 版本治理 + 发布流程”的整体工程体系。

---

## 第七章：常见问题与优化

### 本章学习目标
- 快速识别 Monorepo 常见坑位。
- 掌握性能优化思路。
- 建立可持续演进的维护策略。

### 核心概念解释（通俗版）
Monorepo 像高速公路：车多了就要有匝道规则、限速规则和 ETC（缓存机制），否则再宽也会堵。

### 常见坑位清单
1. **依赖污染**：未声明依赖却能运行（某些工具链“帮你兜底”）。
2. **循环依赖**：包与包互相 import。
3. **构建雪崩**：小改动触发全量构建。
4. **脚本分散**：每个包脚本命名不同，运维难统一。
5. **权限与代码所有权不清**：谁都能改核心包。

### 性能优化思路
1. **按变更范围执行任务**
   - 只测试/构建受影响包。
2. **启用构建缓存（本地 + 远程）**
   - 缓存命中可大幅缩短 CI 时间。
3. **合理拆包，避免“超大核心包”**
   - 核心包太大将导致“改一点，全体重编译”。
4. **依赖收敛与版本统一**
   - 降低 node_modules 冲突与重复安装。
5. **预提交校验最小化**
   - 只对变更文件跑 lint/test，提升开发体验。

### 实战示例（按范围执行）
```bash
# 仅构建受影响应用（示例：只构建 web）
pnpm --filter @book/web build

# 仅测试某个包
pnpm --filter @book/utils test
```

### 常见错误
- 一开始就追求“最复杂工具链”，团队无法维护。
- 没有度量指标（CI 耗时、缓存命中率、失败率），优化无方向。
- 忽略文档，导致规则只存在于“老员工口口相传”。

### 小结
优化不是炫技，目标只有一个：**让团队更快、更稳地交付**。

---

## 第八章：进阶方向与架构思维

### 本章学习目标
- 建立 Monorepo 的长期架构视角。
- 知道何时继续演进，何时保持克制。
- 形成可复用的落地方法论。

### 核心概念解释（通俗版）
架构师思维不是“堆工具”，而是“在成本、效率、风险之间做平衡”。

### 进阶方向
1. **任务编排升级**：接入 Turborepo/Nx，完善缓存和依赖图。
2. **质量门禁**：统一 lint、typecheck、test、e2e 流程。
3. **包可观测性**：记录包变更频率、构建耗时、故障归因。
4. **自动发布体系**：Changesets + CI 自动发版。
5. **架构治理机制**：CODEOWNERS、RFC、依赖规则检查（如 eslint-plugin-boundaries）。

### 实战示例（落地路线图）
```txt
阶段1（1~2周）
- 完成目录迁移到 Monorepo
- 统一 pnpm workspace
- 跑通基础构建与开发命令

阶段2（2~4周）
- 抽离 ui/utils/config 公共包
- 接入 Changesets
- CI 改为按影响范围执行

阶段3（持续）
- 引入缓存与任务编排
- 建立架构治理规范与评审机制
- 持续监控构建效率和质量指标
```

### 常见错误
- 把 Monorepo 当“重构秀场”，一次性大爆炸改造。
- 架构决策不写文档，后续无法维护。
- 工具升级过快，团队学习成本失控。

### 小结
真正的 Monorepo 能力，是“持续演进能力”：先跑通，再规范，再自动化，再治理。

---

## 结语：你现在应该能做到什么
学到这里，你应该已经具备：
1. 解释 Monorepo 与 Multi-repo 的差异和适用场景。
2. 用 pnpm workspace 搭建并运行一个包含 `apps/` + `packages/` 的项目。
3. 设计子包边界、依赖方向、TS 配置与构建流程。
4. 制定基础发布策略并规避常见坑位。
5. 按团队实际情况做性能优化与架构演进。

如果你准备在公司落地，建议按“先小范围试点 -> 固化模板 -> 全团队推广”的节奏推进。
