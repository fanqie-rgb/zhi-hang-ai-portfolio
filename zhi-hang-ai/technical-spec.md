# 职航 AI - 技术方案与系统设计

## 1. 系统架构

```text
Web / Mobile
   │
API Gateway ── Auth / Rate Limit / Audit Log
   │
Workflow Orchestrator
   ├── Job Parser ── schema validation
   ├── Resume Segmenter ── section + sentence spans
   ├── Retrieval Layer ── job taxonomy / policy / examples
   ├── LLM Router ── fast model / quality model / fallback
   ├── Fact Guard ── source span check / numeric check
   └── Evaluation Logger ── trace / latency / token / feedback
   │
Postgres + Object Storage + Vector Index
```

## 2. 结构化输出契约

```json
{
  "job_summary": {
    "goal": "string",
    "keywords": [
      {"name": "用户研究", "priority": "must", "evidence": "需要说明调研对象、方法与结论"}
    ]
  },
  "diagnosis": [
    {
      "source_span": "简历原文片段",
      "status": "match|gap|risk",
      "reason": "与岗位要求的关系",
      "suggestion": "建议动作",
      "needs_confirmation": ["调研人数", "结果指标"]
    }
  ]
}
```

结构化输出用于三件事：前端稳定渲染、字段级事实校验、统计不同类型建议的采纳率。

## 3. Prompt 与模型路由

### Prompt 分层

1. **系统约束层**：角色、事实保护、隐私和拒答边界。
2. **任务层**：岗位解析、诊断、改写、追问分别使用独立模板。
3. **证据层**：只允许引用岗位原文和用户简历片段。
4. **输出层**：强制 JSON Schema，失败时触发修复或降级。

### 路由策略

- 岗位解析与诊断：质量模型，优先稳定性。
- 联想补全与格式化：快速模型，降低延迟和成本。
- 输出校验失败：先进行 schema repair，再失败则返回可解释错误。
- 用户连续追问超过 3 轮：提示补充事实，避免模型自行扩写。

## 4. 事实保护

- 生成前：将用户输入切分为可引用的 source span。
- 生成中：要求每条建议绑定 source_span 或岗位 evidence。
- 生成后：校验数字、公司名、时间和项目名是否出现在来源中。
- 展示时：对“需要补充”的字段使用确认态，不直接写入简历。

## 5. 失败与降级

| 异常 | 用户体验 | 技术处理 |
|---|---|---|
| 岗位文本过短 | 提示补充岗位描述 | 不进入深度诊断 |
| JSON 解析失败 | 保留原文，提示重试 | schema repair + fallback |
| 事实校验失败 | 标记需确认 | 阻止一键采纳 |
| 模型超时 | 展示阶段进度 | 切换快速模型 |

## 6. 成本与可观测性

每次请求记录 trace_id、模型版本、输入输出 token、首 token 延迟、总延迟、重试次数、校验结果和用户反馈。以“每完成一份诊断的成本”作为优化目标，而不是单纯追求单次 token 成本最低。
