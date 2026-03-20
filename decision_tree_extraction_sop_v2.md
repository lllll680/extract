# 决策树抽取SOP（单切片版 - 供glm-4.7执行）

## 任务概述
将一个md切片文档转换为决策树结构，输出一个文件夹。

**输入**：一个md文件，包含标题 + markdown表格
**输出**：一个文件夹，包含：
```
{切片名}/
├── steps/
│   ├── step_0_{name}.yaml      # 头节点
│   ├── step_1_{name}.yaml      # 表格第1行生成的步骤（如有）
│   ├── step_1_1_{name}.yaml    # step_1的子步骤（如需拆分）
│   └── ...
└── workflow.yaml
```

**参考文件目录**：`__FILL_DATA_PATH__`（含steps和workflow格式示例）

---

## 阶段一：读取与理解文档

### 步骤1.1 读取md文件
```
使用file_read读取目标md文件的全部内容。
```

### 步骤1.2 识别文档结构
```
1. 找到文档标题（通常是第一行，如"# 1.6 一键式快速查询 AP 上线失败原因"）
2. 找到入口命令（文档中提到的主要诊断命令，如"display ap online-fail-record"）
3. 找到markdown表格：
   - 表格以 | 开头
   - 第二行是分隔符（如 |---|---|---|）
   - 从第三行开始是数据行
4. 记录表格的列名（表头）
```

### 步骤1.3 创建输出目录
```
1. 根据文档标题生成文件夹名（使用英文或拼音，下划线连接）
2. 在 ./temp/ 下创建该文件夹
3. 在文件夹内创建 steps/ 子目录
```

---

## 阶段二：生成头节点 step_0

头节点是整个决策树的入口，对应文档中的入口命令。

### 步骤2.1 创建 step_0_{name}.yaml

**文件路径**：`{输出目录}/steps/step_0_{name}.yaml`

**字段逐一说明**：

| 字段 | 类型 | 说明 | 填写规则 |
|-----|------|-----|---------|
| `step_id` | string | 步骤唯一标识 | 固定格式：`step_0_{英文名}`，如 `step_0_check_online_fail` |
| `content` | string | 步骤的操作描述 | 从文档中提取该步骤的完整说明文字，保持原文 |
| `type` | string | 步骤类型 | **固定值**：`diagnosis` |
| `view` | string | 视图类型 | **固定值**：`info` |
| `use_skill` | string | 执行的命令 | 从文档中提取的CLI命令，如 `display ap online-fail-record` |
| `inputs` | object | 命令参数 | 如果命令中有变量（如`{ap_id}`），则填入；否则为空对象 `{}` |
| `extraction_schema` | - | 输出提取规则 | **固定为空**（后续人工补充） |
| `transitions` | object | 状态转移规则 | 见下方详细说明 |

**transitions字段结构**：
```yaml
transitions:
  rules:
    # 这里后续根据表格每行填充
    - condition: "extracted.reason == '错误原因1'"
      next_node: step_1_xxx
    - condition: "extracted.reason == '错误原因2'"
      next_node: CONCLUSION_XXX
  on_error:
    handler_execution_failed: CONCLUSION_MANUAL_CHECK
    cli_command_execution_failed: CONCLUSION_MANUAL_CHECK
  default: CONCLUSION_MANUAL_CHECK
```

### 步骤2.2 step_0完整示例
```yaml
step_id: step_0_check_online_fail
content: "执行display ap online-fail-record命令查询AP上线失败原因，根据返回的错误码进行下一步处理。"
type: diagnosis
view: info
use_skill: "display ap online-fail-record"
inputs: {}
extraction_schema:
transitions:
  rules: []
  on_error:
    handler_execution_failed: CONCLUSION_MANUAL_CHECK
    cli_command_execution_failed: CONCLUSION_MANUAL_CHECK
  default: CONCLUSION_MANUAL_CHECK
```

---

## 阶段三：逐行处理表格

**重要**：一次只处理一行，处理完后等待人工确认再继续。

### 步骤3.1 读取表格第i行

```
提取第i行的各列值，根据表头字段名识别：
- condition_column: 哪一列表示"原因/错误码/失败原因"（跳转条件）
- action_column: 哪一列表示"措施/处理建议/解决方法"（目标内容）
- description_column: 哪一列表示"描述/解释/说明"（补充信息，可选）

示例：
| 上线失败原因 | 解释 | 处理建议 |
| In blacklist | AP在黑名单中 | 确认是否需要移出黑名单，如需要执行undo ap blacklist |

提取结果：
- condition_value = "In blacklist"
- action_value = "确认是否需要移出黑名单，如需要执行undo ap blacklist"
- description_value = "AP在黑名单中"
```

### 步骤3.2 判断输出类型

**判断标准**（需要语义理解）：

| action_value特征 | 输出类型 | 判断依据 |
|-----------------|---------|---------|
| 纯描述性语句，描述最终状态或原因 | `CONCLUSION` | 无动词、无操作指示，如"DTLS协商失败"、"License不足" |
| 包含操作指示，需要执行某个动作 | `step` | 有动词，如"执行xxx"、"检查xxx"、"配置xxx" |
| 包含条件分支，如"如果...则..." | `需拆分` | 有条件词，需要拆成多个step |
| 无法判断 | `__NEED_REVIEW__` | 标记后人工处理 |

### 步骤3.3A 若判断为CONCLUSION

**在workflow.yaml的conclusions部分添加**：

```yaml
CONCLUSION_{英文名大写}:
  level: error
  message: "{action_value原文}"
  suggestion: "{根据description_value或上下文推断的建议}"
```

**字段说明**：

| 字段 | 类型 | 说明 | 填写规则 |
|-----|------|-----|---------|
| 键名 | string | 结论唯一标识 | 格式：`CONCLUSION_{英文名大写}`，如 `CONCLUSION_DTLS_FAILED` |
| `level` | string | 严重级别 | 通常填 `error`；若是提示性信息填 `warning`；正常状态填 `info` |
| `message` | string | 结论描述 | 直接使用action_value原文，或description_value |
| `suggestion` | string | 建议措施 | 从文档上下文推断，或填"请联系技术支持" |

**同时更新step_0的transitions.rules**，添加一条规则：
```yaml
- condition: "extracted.reason == '{condition_value}'"
  next_node: CONCLUSION_{英文名大写}
```

### 步骤3.3B 若判断为step

**创建新文件**：`{输出目录}/steps/step_{序号}_{英文名}.yaml`

**字段逐一说明**：

| 字段 | 类型 | 说明 | 填写规则 |
|-----|------|-----|---------|
| `step_id` | string | 步骤唯一标识 | 格式：`step_{序号}_{英文名}`，如 `step_1_remove_blacklist` |
| `content` | string | 步骤操作描述 | 使用action_value的完整原文 |
| `type` | string | 步骤类型 | **固定值**：`diagnosis` |
| `view` | string | 视图类型 | **固定值**：`info` |
| `use_skill` | string | 执行的命令 | 从action_value中提取CLI命令；若无命令则留空 |
| `inputs` | object | 命令参数 | 从命令中提取变量部分，如 `ap_mac: "{ap_mac}"`；若无则为 `{}` |
| `extraction_schema` | - | 输出提取规则 | **固定为空** |
| `transitions` | object | 状态转移规则 | 见下方 |

**transitions填写规则**：

```yaml
transitions:
  rules:
    # 如果这个step还有下一步（从文档可知），填写规则
    # 如果下一步未知，使用占位符
    - condition: "操作成功"
      next_node: __NEED_FILL__   # 占位符，待补充
    # 或者直接跳转到结论
    - condition: "操作完成"
      next_node: CONCLUSION_XXX
  on_error:
    handler_execution_failed: CONCLUSION_MANUAL_CHECK
    cli_command_execution_failed: CONCLUSION_MANUAL_CHECK
  default: CONCLUSION_MANUAL_CHECK
```

**同时更新step_0的transitions.rules**：
```yaml
- condition: "extracted.reason == '{condition_value}'"
  next_node: step_{序号}_{英文名}
```

### 步骤3.3C 若需要拆分

当action_value包含条件分支时（如"确认是否需要xxx，如需要执行yyy"），拆分为多个step：

**拆分示例**：
原文："确认是否需要移出黑名单，如需要执行undo ap blacklist"

拆分为：
1. `step_1_check_blacklist_need` - 检查是否需要移出黑名单
2. `step_1_1_remove_blacklist` - 执行undo ap blacklist（条件成立时）
3. `CONCLUSION_BLACKLIST_OK` - 无需处理（条件不成立时）

**step_1文件**：
```yaml
step_id: step_1_check_blacklist_need
content: "确认AP是否需要移出黑名单"
type: diagnosis
view: info
use_skill: ""
inputs: {}
extraction_schema:
transitions:
  rules:
    - condition: "need_remove == true"
      next_node: step_1_1_remove_blacklist
    - condition: "need_remove == false"
      next_node: CONCLUSION_BLACKLIST_OK
  on_error:
    handler_execution_failed: CONCLUSION_MANUAL_CHECK
    cli_command_execution_failed: CONCLUSION_MANUAL_CHECK
  default: CONCLUSION_MANUAL_CHECK
```

**step_1_1文件**：
```yaml
step_id: step_1_1_remove_blacklist
content: "执行undo ap blacklist命令将AP移出黑名单"
type: diagnosis
view: info
use_skill: "undo ap blacklist"
inputs:
  ap_mac: "{ap_mac}"
extraction_schema:
transitions:
  rules: []
  on_error:
    handler_execution_failed: CONCLUSION_MANUAL_CHECK
    cli_command_execution_failed: CONCLUSION_MANUAL_CHECK
  default: CONCLUSION_MANUAL_CHECK
```

### 步骤3.4 人工确认

处理完当前行后，向用户展示：

```
===== 第 {i} 行处理结果 =====

【原始数据】
- 条件列: {condition_value}
- 措施列: {action_value}
- 描述列: {description_value}

【判断结果】
- 类型: {CONCLUSION / step / 需拆分}

【生成内容】
{展示生成的yaml内容}

【已更新】
- step_0的transitions.rules已添加新规则

请确认：[确认继续] / [需要修改] / [跳过此行]
```

---

## 阶段四：生成workflow.yaml

处理完所有表格行后，汇总生成workflow.yaml。

**文件路径**：`{输出目录}/workflow.yaml`

**字段逐一说明**：

| 字段 | 类型 | 说明 | 填写规则 |
|-----|------|-----|---------|
| `workflow_id` | string | 工作流唯一标识 | 格式：`wf_{英文名}`，如 `wf_ap_online_fail` |
| `name` | string | 工作流名称 | 使用文档标题，如 "AP上线失败原因诊断" |
| `version` | string | 版本号 | 可留空或填 `"1.0"` |
| `input_schema` | object | 输入参数定义 | 见下方模板 |
| `start_node` | string | 起始节点 | 填step_0的step_id，如 `step_0_check_online_fail` |
| `steps` | array | 所有步骤列表 | 列出所有生成的step_id |
| `conclusions` | object | 所有结论定义 | 包含所有CONCLUSION_xxx |

**input_schema模板**（通常不变）：
```yaml
input_schema:
  type: object
  properties:
    task_id:
      type: string
      title: 任务ID
    task_name:
      type: string
      title: 任务名称
  required:
    - task_id
    - task_name
```

**workflow.yaml完整示例**：
```yaml
workflow_id: wf_ap_online_fail
name: "AP上线失败原因诊断"
version: "1.0"
input_schema:
  type: object
  properties:
    task_id:
      type: string
      title: 任务ID
    task_name:
      type: string
      title: 任务名称
  required:
    - task_id
    - task_name
start_node: step_0_check_online_fail
steps:
  - step_0_check_online_fail
  - step_1_remove_blacklist
  - step_1_1_execute_undo
  - step_2_check_license
conclusions:
  CONCLUSION_MANUAL_CHECK:
    level: warning
    message: "需要人工检查"
    suggestion: "请联系技术支持进行进一步排查"
  CONCLUSION_DTLS_FAILED:
    level: error
    message: "DTLS协商失败"
    suggestion: "检查证书配置是否正确"
  CONCLUSION_LICENSE_INSUFFICIENT:
    level: error
    message: "License数量不足"
    suggestion: "申请更多AP License"
```

---

## 命名规范速查

| 类型 | 命名格式 | 示例 |
|-----|---------|------|
| 主步骤 | `step_{n}_{英文名}` | `step_1_check_blacklist` |
| 子步骤 | `step_{n}_{m}_{英文名}` | `step_1_1_remove_blacklist` |
| 更深子步骤 | `step_{n}_{m}_{k}_{英文名}` | `step_1_1_1_verify_result` |
| 结论 | `CONCLUSION_{英文名大写}` | `CONCLUSION_DTLS_FAILED` |
| 工作流 | `wf_{英文名}` | `wf_ap_online_fail` |
| 文件夹 | `{英文名或拼音}` | `ap_online_fail_diagnosis` |

---

## 固定模板（直接复制）

**step通用固定字段**：
```yaml
type: diagnosis
view: info
extraction_schema:
transitions:
  on_error:
    handler_execution_failed: CONCLUSION_MANUAL_CHECK
    cli_command_execution_failed: CONCLUSION_MANUAL_CHECK
  default: CONCLUSION_MANUAL_CHECK
```

**必须存在的conclusion**：
```yaml
CONCLUSION_MANUAL_CHECK:
  level: warning
  message: "需要人工检查"
  suggestion: "请联系技术支持进行进一步排查"
```

---

## 检查清单

完成后检查：

- [ ] 每个step_id在steps列表中都有记录
- [ ] 每个CONCLUSION_xxx在conclusions中都有定义
- [ ] step_0的transitions.rules包含了表格所有行的条件
- [ ] 所有`__NEED_FILL__`占位符已标记待补充
- [ ] 所有`__NEED_REVIEW__`已人工复核
- [ ] use_skill中的命令正确提取
- [ ] inputs中的参数与命令中的变量对应