# Skill: 评测 Skill 输出报告

## 触发条件

当用户要求评测 Skill 分析报告的质量时触发，例如：
- "评测 reports 目录下的报告"
- "对比 report 和 standard"
- "检查 Skill 输出的正确率"

## 评价维度

用户会围绕以下**六个方向**提问，评测时需按方向逐一判定：

| 方向 | 用户问题 | 标签 |
|------|----------|------|
| Q1 | 哪些模块没有被业务代码实际使用，只是因为副作用被保留进 bundle？ | `[Q1:SIDE_EFFECTS]` |
| Q2 | 从产物体积角度看，当前 bundle 最大的优化点有哪些？ | `[Q2:OUTPUT_OPT]` |
| Q2 | 请做一次 bundle 产物优化诊断，覆盖重复依赖、chunk 冗余、副作用和大资源。 | `[Q2:OUTPUT_OPT]` |
| Q3 | 请列出重复包的包名、版本、大小和涉及 chunk。 | `[Q3:DUPLICATE_PKGS]` |
| Q4 | 当前有哪些 tree-shaking 失效点？应该怎么优化？ | `[Q4:TREE_SHAKING]` |
| Q4 | 哪些模块因为导入方式、barrel file 或 CJS 导致没有被摇掉？ | `[Q4:TREE_SHAKING]` |
| Q5 | 哪些 loader 耗时最高？分别处理了哪些文件或资源？ | `[Q5:LOADER_COST]` |
| Q5 | 当前构建里 loader 阶段的主要性能瓶颈是什么？ | `[Q5:LOADER_COST]` |
| Q6 | 哪些 plugin hook 耗时最高？耗时集中在哪些阶段？ | `[Q6:PLUGIN_COST]` |
| Q6 | 当前构建里 plugin 阶段的主要性能瓶颈是什么？ | `[Q6:PLUGIN_COST]` |

> 不是每个 case 都涉及全部六个方向，如果某方向在某 case 中无对应检查点则跳过。

## 任务

将 `eval/reports/<case>.md`（Skill 实际输出）与 `eval/standards/<case>.md`（标准答案）逐条对比，**按六个问题方向**判定每个检查点是否被覆盖，输出正确率和错误率。

## 执行步骤

### Step 1: 读取所有 standard 文件

读取 `eval/standards/` 目录下的所有 `.md` 文件。每个文件按问题方向（`## Q1` ~ `## Q6`）组织，其中用 `### [CHECK]` 标记的段落是一个独立检查点。

### Step 2: 读取对应的 report 文件

对每个 standard 文件（如 `standards/ui-components.md`），读取对应的 report 文件（`reports/ui-components.md`）。如果 report 文件不存在则跳过该 case。

### Step 3: 按方向逐条判定

对每个问题方向下的 `[CHECK]` 检查点，判断 report 中是否**覆盖**了该检查点：

**判定规则：**
- "覆盖"是指 report 中提到了与检查点相同的概念，不要求完全一样的措辞
- 语义等价即可，但必须**清晰明确**地传达了检查点的核心观点
- 如果 report 只是模糊提及或部分覆盖，判定为**未覆盖**
- 严格判定：宁可判为未覆盖，也不要放松标准

对每个检查点输出：
- `[PASS]` 或 `[FAIL]`
- 简要理由（一句话）

### Step 4: 汇总结果

输出格式：

```
## 评测结果

### <case_name>（X/Y）

#### Q1: 仅因副作用而被打包的文件
- [PASS] 检查点标题 — 理由
- [FAIL] 检查点标题 — 理由

#### Q2: 产物优化
- [PASS] 检查点标题 — 理由
- ...

#### Q3: 重复包
- ...

#### Q4: tree-shaking 优化
- ...

#### Q5: loader 耗时
- ...

#### Q6: plugin 耗时
- ...

正确率: X/Y = XX.X%

---

### <case_name>（X/Y）
...

---

## 总计

- 总检查点: N
- 通过: M
- 正确率: M/N = XX.X%
- 错误率: (N-M)/N = XX.X%

### 按方向统计

| 方向 | 总计 | 通过 | 正确率 |
|------|------|------|--------|
| Q1: 副作用文件 | a | b | b/a |
| Q2: 产物优化 | c | d | d/c |
| Q3: 重复包 | e | f | f/e |
| Q4: tree-shaking | g | h | h/g |
| Q5: loader 耗时 | i | j | j/i |
| Q6: plugin 耗时 | k | l | l/k |
```

### Step 5: 保存结果

将上述评测结果保存到 `eval/results/<run_name>.md` 文件中。如果用户没有指定 run_name，使用 `v1`、`v2` 等自增命名。

## 注意事项

- 每个检查点独立判定，不要因为前一个通过了就放松后续的标准
- report 中可能包含标准答案中没有提到的额外发现，这不影响判定，只关注 standard 中列出的检查点是否被覆盖
- 如果 report 文件为空或内容明显不完整，所有检查点都判为 FAIL
- 某些问题方向在某些 case 中可能不适用（无对应检查点），跳过即可
