# logicFlow — 宜搭集成&自动化（逻辑流）技能

本技能用于在宜搭平台创建「集成&自动化」（逻辑流），支持场景：**表单事件触发 → 获取单条数据（可选）→ 钉钉工作通知**。

## 功能概述

- 监听指定表单的新增 / 更新 / 删除 / 评论事件
- 可选：从另一张表单（B 表单）获取单条数据，支持按触发表单字段值过滤
- 通知内容和标题支持引用表单字段变量（`#{fieldId-ComponentType}#` 格式）
- 支持保存为草稿（未开启状态）或直接发布（开启状态）

## 命令格式

```bash
openyida logicflow create <appType> <formUuid> <flowName> [选项]
```

### 参数说明

| 参数 | 必填 | 说明 |
| --- | --- | --- |
| `appType` | 是 | 应用 ID，如 `APP_XXXX` |
| `formUuid` | 是 | 触发表单 UUID，如 `FORM-XXXX` |
| `flowName` | 是 | 逻辑流名称 |

### 选项说明

| 选项 | 默认值 | 说明 |
| --- | --- | --- |
| `--process-code <code>` | 自动生成 | 已有逻辑流的 processCode（`LPROC-xxx` 格式），不传则自动生成 |
| `--receivers <userId,...>` | 空（无接收人） | 接收钉钉工作通知的用户 ID，多个用逗号分隔 |
| `--title <title>` | 同 flowName | 通知标题，支持 `#{fieldId-ComponentType}#` 引用表单字段 |
| `--content <content>` | `"表单有新记录提交，请及时查看。"` | 通知内容，支持 `#{fieldId-ComponentType}#` 引用表单字段 |
| `--events <insert,update>` | `insert` | 触发事件，可选值：`insert`/`update`/`delete`/`comment`（也支持别名 `create`），多个用逗号分隔 |
| `--data-form-uuid <formUuid>` | 不启用 | 获取单条数据节点的目标表单 UUID（B 表单），传入后在触发节点和通知节点之间插入 GetSingleDataNode |
| `--data-condition <bFieldId:bFieldName:aFieldId[:componentType]>` | 无 | 获取单条数据的过滤条件，可多次传入；格式：`B表单字段ID:B表单字段名:A表单字段ID[:组件类型]`，组件类型默认 `TextField` |
| `--publish` | 不发布 | 加此标志则保存后立即发布（开启状态），否则仅保存为草稿 |

### 示例

```bash
# 最简用法：表单新增时通知指定用户，仅保存草稿
openyida logicflow create APP_XXX FORM-XXX "新增记录通知" \
  --receivers user123 \
  --title "有新记录提交" \
  --content "表单有新记录提交，请及时处理。"

# 引用表单字段变量，保存并发布
openyida logicflow create APP_XXX FORM-XXX "记录变更通知" \
  --receivers user123,user456 \
  --title "记录变更：#{textField_abc-TextField}#" \
  --content "内容：#{textField_abc-TextField}#" \
  --events insert,update,delete,comment \
  --publish

# 带获取单条数据节点：触发时从 B 表单获取匹配记录，再发送通知
# --data-condition 格式：B表单字段ID:B表单字段名:A表单字段ID[:组件类型]
# 可多次传入 --data-condition 添加多个过滤条件
openyida logicflow create APP_XXX FORM-A-XXX "跨表通知" \
  --receivers user123 \
  --title "关联记录变更：#{textField_a1-TextField}#" \
  --content "B表单数据已更新，请查看。" \
  --events insert,update \
  --data-form-uuid FORM-B-XXX \
  --data-condition "textField_b1:B表单姓名字段:textField_a1:TextField" \
  --publish
```

## 字段变量引用格式

在通知标题和内容中，可以使用 `#{fieldId-ComponentType}#` 格式引用触发表单的字段值：

```
#{textField_mmq4ldti-TextField}#
#{numberField_abc123-NumberField}#
#{selectField_xyz-SelectField}#
```

- `fieldId`：字段 ID（可通过 `yida-get-schema` 技能查询）
- `ComponentType`：字段组件类型（如 `TextField`、`NumberField`、`SelectField` 等）

## 输出结果

命令执行成功后，向 stdout 输出 JSON：

```json
{
  "success": true,
  "published": false,
  "processCode": "LPROC-XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX",
  "flowName": "新增记录通知",
  "appType": "APP_XXX",
  "formUuid": "FORM-XXX",
  "formEventTypes": ["insert"]
}
```

加 `--publish` 后 `published` 为 `true`。若发布失败，`published` 为 `false` 并附带 `warning` 字段说明原因。

## 调用流程

1. 读取 `.cache/cookies.json` 获取登录态（不存在则触发扫码登录）
2. 生成 `processCode`（`LPROC-xxx` 格式）和各节点 ID（`node_xxx` 格式）
3. 构建 `json` 参数（节点定义）和 `viewJson` 参数（画布 Schema）
4. 调用 `saveProcess` 接口（`isOnline=false`）保存为草稿
5. 若指定 `--publish`，再次调用 `saveProcess` 接口（`isOnline=true`）发布生效

## 逻辑流节点结构

本技能生成的逻辑流支持两种节点链路：

**不带获取单条数据（3 个节点）：**
```
StartNode（表单事件触发）
    ↓
SendMessageNode（钉钉工作通知）
    ↓
EndNode（结束）
```

**带获取单条数据（4 个节点，传入 `--data-form-uuid` 时）：**
```
StartNode（表单事件触发）
    ↓
GetSingleDataNode（获取单条数据，从 B 表单按条件查询）
    ↓
SendMessageNode（钉钉工作通知）
    ↓
EndNode（结束）
```

### json 参数中的节点类型（type 字段）

| 节点 | type 值 | viewJson 中的 componentName |
| --- | --- | --- |
| 触发节点 | `trigger` | `StartNode` |
| 获取单条数据节点（可选） | `dataRetrieve` | `GetSingleDataNode` |
| 消息通知节点 | `sendMessage` | `SendMessageNode` |
| 结束节点 | `finish` | `EndNode` |

### StartNode（触发节点）关键属性

| 属性路径 | 说明 |
| --- | --- |
| `props.inputs.formEventType` | 触发事件列表：`["insert","update","delete","comment"]` |
| `props.inputs.formUuid` | 触发表单 UUID |
| `props.inputs.triggerFormEventRecursively` | 固定 `true` |
| `props.triggerType` | 固定 `"FormEvent"` |

### GetSingleDataNode（获取单条数据节点）关键属性

> 仅在传入 `--data-form-uuid` 时生成此节点，插入在触发节点和通知节点之间。

**json 参数中的 props：**

| 属性路径 | 说明 |
| --- | --- |
| `props.type` | 固定 `"single"` |
| `props.filterType` | 固定 `"condition"` |
| `props.sourceId` | B 表单 UUID（`--data-form-uuid` 传入的值） |
| `props.appType` | 应用 appType |
| `props.originalType` | 固定 `"form"` |
| `props.condition` | 过滤条件对象（见下方 condition 结构） |
| `props.quantity` | 固定 `"1"`（字符串） |
| `props.dataRules` | 固定结构，包含一条空规则 |
| `props.assignments` | 固定 `[]` |

**viewJson 参数中的 props（GetSingleDataNode）：**

| 属性路径 | 说明 |
| --- | --- |
| `props.nodeName` | 固定 `"GetSingleDataNode"` |
| `props.name` | 固定 `"获取单条数据"` |
| `props.getData.sourceId` | B 表单 UUID |
| `props.getData.targetItem.formItem.formUuid` | B 表单 UUID |
| `props.getData.condition` | 与 json 参数中的 condition 结构相同 |
| `props.getData.quantity` | 固定 `1`（数字） |
| `props.getData.rulesFilter` | 固定 `[]`（脚本不填充可用字段列表） |
| `props.getData.outputs` | 固定 `[]`（脚本不填充输出字段列表） |

**condition 对象结构（来自真实抓包）：**

```json
{
  "condition": "AND",
  "rules": [
    {
      "id": "B表单字段ID",
      "op": "包含",
      "operators": [],
      "value": "A表单字段ID（processVar引用）",
      "componentType": "TextField",
      "ruleId": "item-xxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
      "parentId": "group-xxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
      "extValue": "processVar",
      "ruleValue": "A表单字段ID",
      "name": "B表单字段名",
      "valueType": "processVar",
      "ruleType": "rule_text",
      "opCode": "Contain"
    }
  ],
  "ruleId": "group-xxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "conditionCode": "&&"
}
```

- `ruleId`（rule 级别）格式：`item-` + UUID 格式随机串
- `ruleId`（group 级别）格式：`group-` + UUID 格式随机串
- `parentId` 与 group 的 `ruleId` 相同

### SendMessageNode（消息通知节点）关键属性

| 属性路径 | 说明 |
| --- | --- |
| `props.messageType` | 固定 `"NORMAL"` |
| `props.messageInfo.title` | 通知标题，支持 `#{fieldId-ComponentType}#` |
| `props.messageInfo.content` | 通知内容，支持 `#{fieldId-ComponentType}#` |
| `props.messageInfo.buttons` | 按钮列表，`type: "commit"` 为查看详情，`type: "custom"` 为自定义链接 |
| `props.toUsers` | 接收人列表，格式：`[{"userId":"xxx","userName":"yyy"}]` |
| `props.userFields` | 通过字段指定接收人，如 `["form_inst_modifier"]`（最后修改者） |

## 接口说明

### saveProcess（保存 / 发布逻辑流）

> 保存和发布使用**同一个接口**，通过 `isOnline` 参数区分。

- **地址**：`POST /alibaba/web/{appType}/query/simpleProcess/saveProcess.json`
- **Content-Type**：`application/x-www-form-urlencoded`
- **参数**：

| 参数 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `_csrf_token` | String | 是 | CSRF Token |
| `formUuid` | String | 是 | 关联表单 UUID |
| `isLogic` | String | 是 | 固定 `"true"` |
| `isOnline` | String | 是 | `"false"`=保存草稿（未开启），`"true"`=发布生效（已开启） |
| `json` | String (JSON) | 是 | 节点定义 JSON 字符串（见下方结构说明） |
| `viewJson` | String (JSON) | 是 | 画布 Schema JSON 字符串（见下方结构说明） |
| `processCode` | String | 是 | 逻辑流唯一标识，格式 `LPROC-xxx`（自动生成或由用户传入） |
| `needReportLine` | String | 是 | 固定 `"y"` |

- **返回值**：

```json
{ "success": true }
```

### json 参数结构（节点定义）

```json
{
  "props": {
    "allowWithdraw": true,
    "allowCollaboration": true,
    "allowTemporaryStorage": true,
    "processCode": "LPROC-xxx"
  },
  "nodes": [
    {
      "name": { "en_US": "Form event trigger", "zh_CN": "表单事件触发", "type": "i18n" },
      "description": "",
      "type": "trigger",
      "nodeId": "node_xxx",
      "prevId": "",
      "nextId": ["node_yyy"],
      "props": {
        "inputs": {
          "formEventType": ["insert", "update", "delete", "comment"],
          "formUuid": "FORM-xxx",
          "conditions": null,
          "activityAction": [],
          "triggerFormEventRecursively": true
        },
        "triggerType": "FormEvent"
      },
      "childNodes": []
    },
    {
      "name": { "zh_CN": "消息通知", "en_US": "" },
      "description": "请设置消息通知",
      "type": "sendMessage",
      "nodeId": "node_yyy",
      "prevId": "",
      "nextId": ["node_zzz"],
      "props": {
        "template": { "templateName": "" },
        "messageType": "NORMAL",
        "messageInfo": {
          "title": "通知标题（支持 #{fieldId-ComponentType}#）",
          "content": "通知内容（支持 #{fieldId-ComponentType}#）",
          "buttons": [
            {
              "name": "查看详情",
              "type": "commit",
              "value": "//yidalogin.aliwork.com/{appType}/formDetail/{formUuid}?formInstId=${formInstId}",
              "buttonUuid": "button-xxx"
            }
          ]
        },
        "appType": "APP_xxx",
        "toRoles": [],
        "toUsers": [{ "userId": "xxx", "userName": "yyy" }],
        "userFields": []
      },
      "childNodes": []
    },
    {
      "name": { "en_US": "end", "zh_CN": "结束", "type": "i18n" },
      "description": "",
      "type": "finish",
      "nodeId": "node_zzz",
      "prevId": "",
      "nextId": [],
      "props": {},
      "childNodes": []
    }
  ]
}
```

### viewJson 参数结构（画布 Schema）

```json
{
  "schema": {
    "componentName": "CanvasEngine",
    "id": "node_canvas",
    "props": {},
    "children": [
      {
        "componentName": "StartNode",
        "id": "node_xxx",
        "props": {
          "nodeName": "StartNode",
          "name": { "en_US": "Form event trigger", "zh_CN": "表单事件触发", "type": "i18n" },
          "start": {
            "examineApproveType": "processFinish",
            "formEventType": ["insert", "update", "delete", "comment"],
            "dataFilterType": "all",
            "fieldType": "all",
            "conditions": { "condition": "AND", "rules": [] },
            "formUuid": "FORM-xxx",
            "triggerType": "FormEvent",
            "type": "form",
            "triggerFormEventRecursively": true,
            "examineApproveNode": "",
            "examineApproveActiveList": []
          }
        }
      },
      {
        "componentName": "SendMessageNode",
        "id": "node_yyy",
        "props": {
          "nodeName": "SendMessageNode",
          "name": "消息通知",
          "description": "请设置消息通知",
          "sendMessageRules": {
            "template": { "templateName": "" },
            "messageType": "NORMAL",
            "messageInfo": {
              "title": "通知标题",
              "content": "通知内容",
              "buttons": [...]
            },
            "appType": "APP_xxx",
            "toRoles": [],
            "toUsers": [{ "userId": "xxx", "userName": "yyy" }],
            "userFields": []
          }
        }
      },
      {
        "componentName": "EndNode",
        "id": "node_zzz",
        "props": {
          "name": { "en_US": "end", "zh_CN": "结束", "type": "i18n" }
        }
      }
    ],
    "globalSetting": {}
  }
}
```

### listflow（查询逻辑流列表）

- **地址**：`GET /alibaba/web/{appType}/query/appLogicflowBinding/listflow.json`
- **参数**：

| 参数 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `_csrf_token` | String | 是 | CSRF Token |
| `appType` | String | 是 | 应用 ID |
| `type` | Number | 否 | 触发类型：`1`=表单事件触发，不传=全部 |
| `formUuid` | String | 否 | 按触发表单过滤 |
| `status` | String | 否 | 状态过滤：`y`=已开启，`n`=已关闭，不传=全部 |
| `key` | String | 否 | 名称关键字搜索 |
| `pageIndex` | Number | 否 | 页码，默认 `1` |
| `pageSize` | Number | 否 | 每页数量，默认 `10` |

- **返回值**：

```json
{
  "success": true,
  "content": {
    "data": [...],
    "totalCount": 0
  }
}
```

### switchflow（开启/关闭逻辑流）

> ⚠️ 注意：此接口命名空间为 `formLogicflowBinding`，与 `listflow` 的 `appLogicflowBinding` 不同。

- **地址**：`POST /alibaba/web/{appType}/query/formLogicflowBinding/switchflow.json`
- **Content-Type**：`application/x-www-form-urlencoded`
- **参数**：

| 参数 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `_csrf_token` | String | 是 | CSRF Token |
| `processCode` | String | 是 | 逻辑流唯一标识，格式 `LPROC-xxx` |
| `formUuid` | String | 是 | 关联表单 UUID |
| `type` | Number | 是 | 触发类型：`1`=表单事件触发 |
| `enable` | String | 是 | `y`=开启，`n`=关闭 |

- **返回值**：

```json
{ "success": true }
```

## 前置依赖

- Node.js
- 项目根目录存在 `.cache/cookies.json`（首次运行会自动触发扫码登录）

## 文件结构

```
lib/
└── create-logicflow.js    # logicflow 命令实现
```

## 与其他技能配合

1. **创建应用** → 使用 `yida-create-app` 技能获取 `appType`
2. **创建表单页面** → 使用 `yida-create-form-page` 技能获取 `formUuid`
3. **查询字段 ID** → 使用 `yida-get-schema` 技能获取 `fieldId` 和 `ComponentType`，用于构建字段变量引用
4. **创建集成&自动化** → 本技能，传入 `appType` 和 `formUuid`

## 注意事项

- `--receivers` 填写的是宜搭/钉钉用户 ID（`userId`），不是姓名
- 触发事件使用 API 内部名称：`insert`（新增）、`update`（更新）、`delete`（删除）、`comment`（评论），也支持别名 `create`
- `processCode` 格式为 `LPROC-` 加 38 位大写字母数字，不传则自动随机生成
- 保存（草稿）和发布使用**同一个接口** `saveProcess`，通过 `isOnline` 参数区分
- 错误码处理：接口返回 `errorCode: "TIANSHU_000030"`（csrf 校验失败）时，脚本会自动刷新 token 后重试；`errorCode: "307"`（登录过期）时，会自动重新登录后重试
