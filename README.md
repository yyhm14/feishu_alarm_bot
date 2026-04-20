# feishu-alarm-bot — 飞书报警机器人开发助手

> 一个 Claude Code Skill，帮助你快速开发、审查、横向复制**飞书消息卡片类报警机器人**项目。

## 适用项目类型

本 Skill 针对以下架构的项目提供专项支持：

```
外部设备（SCADA / PLC / VBS 脚本）
        │  HTTP POST
        ▼
   Flask 接口（接收报警数据）
        │
   防重复过滤 ──── AlarmDeduplicator
        │
   飞书卡片推送 ── im.v1.message.create
        │            用户在卡片填写表单
        ▼
   WebSocket 回调 ── P2CardActionTrigger
        │
   多维表格写入 ─── bitable.v1.app_table_record.create
        │
   统计持久化 + 定时日报（DailyStatistics）
```

**典型业务场景**：
- 工厂设备故障报警推送
- SCADA/PLC 异常通知
- 生产线质量问题记录
- 仓库库存预警
- 任何"设备报警 → 飞书推送 → 人工确认 → 记录归档"的流程

---

## 命令列表

### `/bot-review [file] [--focus=all]`

审查飞书机器人代码，识别常见陷阱并给出修复方案。

**检查维度**：

| 维度 | 典型问题 |
|------|---------|
| **API 使用** | 配置重复定义、响应未校验、content 未 json.dumps |
| **并发安全** | 全局变量无锁、Flask 多线程共享状态 |
| **可靠性** | 重启数据丢失、相对路径在定时任务中失效 |
| **统计逻辑** | 8:00~次日8:00 时间段计算错误 |
| **卡片结构** | schema 版本、update_multi 缺失、动态 name 表单串扰 |

```bash
# 审查整个项目
/bot-review

# 审查特定文件
/bot-review alarm_service.py

# 只看并发和可靠性问题
/bot-review --focus=reliability
```

---

### `/bot-scaffold <scenario> [--modules=all]`

根据业务描述生成完整的项目脚手架代码。

**支持场景**：

| 示例 scenario | 生成的字段结构 |
|--------------|--------------|
| `设备故障报警` | 线体、设备、故障信息、处理人、类别、原因、措施 |
| `生产异常通知` | 产线、工位、异常类型、影响数量、处置方式 |
| `库存预警` | 物料编码、仓库、当前库存、安全库存、负责人 |
| `质量问题` | 产品、批次、缺陷类型、发现工序、处置结果 |

**可选模块**：

| module | 内容 |
|--------|------|
| `push` | Flask 路由 + `push_alarm_card` 函数 |
| `card` | `P2CardActionTrigger` 回调 + 卡片构建 |
| `bitable` | 多维表格写入函数（含字段映射） |
| `dedup` | `AlarmDeduplicator` 防重复类 |
| `report` | `DailyStatistics` + 定时日报 |
| `all` | 以上全部（默认） |

```bash
# 生成设备故障报警完整项目
/bot-scaffold "设备故障报警"

# 只生成推送和防重复模块
/bot-scaffold "库存预警" --modules=push,dedup

# 生成生产记录表单项目
/bot-scaffold "生产记录录入" --modules=card,bitable
```

---

### `/bot-card <type> [--fields=字段列表]`

生成飞书消息卡片 JSON 结构（Python dict 函数形式）。

| type | 用途 |
|------|------|
| `alarm` | 报警卡片（含处理表单，红色 header） |
| `success` | 已处理卡片（绿色 header，替换原报警卡片） |
| `report` | 日报统计卡片（蓝色 header，含推送/填写率） |
| `form` | 手动录入表单卡片（回响模式，保留上次配置） |
| `custom` | 根据 fields 参数动态生成 |

```bash
# 生成报警卡片
/bot-card alarm --fields="线体,设备,故障信息,报警时间"

# 生成日报卡片
/bot-card report

# 生成自定义卡片
/bot-card custom --fields="物料编码,仓库,当前库存,安全库存"
```

---

## 安装方法



### 方法二：从 GitHub 直接安装

```bash
claude plugin install https://github.com/yyhm14/feishu-alarm-bot
```




## 技术背景：关键 API 速查

### 1. 消息推送

```python
from lark_oapi.api.im.v1 import CreateMessageRequest, CreateMessageRequestBody

request = CreateMessageRequest.builder() \
    .receive_id_type("chat_id")          # chat_id / open_id / user_id
    .request_body(
        CreateMessageRequestBody.builder()
            .receive_id("oc_xxx")
            .msg_type("interactive")      # 卡片必须是 interactive
            .content(json.dumps(card))   # ← 必须是字符串
            .build()
    ).build()

response = api_client.im.v1.message.create(request)
if not response.success():
    print(f"推送失败: {response.code} - {response.msg}")
```

### 2. 多维表格写入

```python
from lark_oapi.api.bitable.v1 import CreateAppTableRecordRequest, AppTableRecord

fields = {
    "故障信息": "伺服报警",
    "处理人": [{"id": "ou_xxx"}],   # 人员字段：列表包 dict
    "推送时间": "2024-03-30 10:00:00"
}
request = CreateAppTableRecordRequest.builder() \
    .app_token("VBRqxxx") \
    .table_id("tblXxx") \
    .request_body(AppTableRecord.builder().fields(fields).build()) \
    .build()

response = api_client.bitable.v1.app_table_record.create(request)
```

### 3. 卡片交互回调

```python
from lark_oapi.event.callback.model.p2_card_action_trigger import (
    P2CardActionTrigger, P2CardActionTriggerResponse
)

def do_card_action_trigger(data: P2CardActionTrigger) -> P2CardActionTriggerResponse:
    action_value = data.event.action.value or {}
    if isinstance(action_value, str):
        action_value = json.loads(action_value)

    form_value = getattr(data.event.action, "form_value", {}) or {}
    operator_id = data.event.operator.open_id

    if action_value.get("action") == "complete_alarm":
        # 处理表单提交
        return P2CardActionTriggerResponse({
            "toast": {"type": "success", "content": "提交成功"},
            "card": {"type": "raw", "data": new_card_dict}
        })

    return P2CardActionTriggerResponse({})
```

### 4. WebSocket 长连接启动

```python
import lark_oapi as lark

event_handler = (
    lark.EventDispatcherHandler.builder(VERIFICATION_TOKEN, ENCRYPT_KEY)
    .register_p2_card_action_trigger(do_card_action_trigger)
    .build()
)

# 长连接，无需公网 IP
lark.ws.Client(APP_ID, APP_SECRET,
               event_handler=event_handler,
               log_level=lark.LogLevel.INFO).start()
```

---

## 常见陷阱总结

| 陷阱 | 现象 | 解决方案 |
|------|------|---------|
| 配置重复定义 | 两套 BITABLE_TOKEN 指向不同表 | 统一在文件顶部定义一次 |
| content 未 json.dumps | 推送 API 报参数错误 | `content=json.dumps(card_dict)` |
| 相对路径 | 定时任务启动时文件写入失败 | `BASE_DIR = os.path.dirname(os.path.abspath(__file__))` |
| 动态 name 表单串扰 | 多张卡片的表单数据互相覆盖 | 每张卡片的表单 name 加 uuid 后缀 |
| 日报时间段算错 | 0:00~7:59 被统计到"今天" | `if now.hour < 8: return yesterday` |
| update_multi 缺失 | 群内多人操作互相覆盖 | `"config": {"update_multi": True}` |
| 全局变量无锁 | Flask 多线程下数据竞争 | 用 `threading.Lock()` 保护 |

---

## 飞书后台配置清单

1. **创建应用** → 飞书开放平台 → 企业自建应用
2. **开启权限**：
   - `im:message`（发送消息）
   - `bitable:app`（多维表格读写）
   - `contact:user.base:readonly`（获取用户名，可选）
3. **启用长连接**：应用功能 → 机器人 → 消息与事件，开启 WebSocket 长连接
4. **订阅事件**：无需配置，`P2CardActionTrigger` 通过 SDK 自动注册
5. **添加到群**：将机器人添加到目标群聊，获取群的 `chat_id`（`oc_xxx`）

---

## 版本历史

| 版本 | 更新内容 |
|------|---------|
| v1.0.0 | 初始版本：bot-review、bot-scaffold、bot-card 三个命令 |

---

## 作者

weichao13

## 许可证

MIT License
