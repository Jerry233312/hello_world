# 医疗模拟游戏 RAG 后端

基于 RAG（检索增强生成）和大语言模型的医疗诊断模拟系统。医生通过与 AI 患者交互进行诊断，系统通过关键信息触发机制逐步披露病历信息，最后对诊断结果进行智能评分。

## 核心特性

### 1. 自由对话接口
- 医生可以自然语言与 AI 患者进行对话
- 支持多轮对话，保持对话上下文
- 患者根据性格特征做出合理回应

### 2. 关键信息触发机制
- **渐进式信息披露**：只有问到特定关键词才返回相应病历信息
- **可配置触发词**：支持自定义不同类型的触发关键词
- **信息状态追踪**：记录哪些信息已被触发

### 3. 智能诊断评分
- **多维度评估**：
  - 主诊断匹配度（权重 50%）
  - 鉴别诊断覆盖度（权重 20%）
  - 关键发现识别度（权重 30%）
- **详细反馈**：基于评分提供针对性改进建议
- **LLM 辅助分析**：使用大模型深度分析诊断推理过程

## 系统架构

```
┌─────────────────────────────────────────┐
│         FastAPI 后端服务                │
├─────────────────────────────────────────┤
│  ┌──────────┬──────────┬──────────┐    │
│  │ 对话路由 │ 模拟路由 │ 诊断路由 │    │
│  └──────────┴──────────┴──────────┘    │
├─────────────────────────────────────────┤
│  ┌─────────────┬─────────────┐          │
│  │ LLM Service │ Vector DB   │          │
│  │ (OpenAI)    │ (Pinecone)  │          │
│  └─────────────┴─────────────┘          │
└─────────────────────────────────────────┘
```

## 快速开始

### 环境配置

1. 克隆仓库并进入目录
2. 创建虚拟环境：
```bash
python -m venv venv
source venv/bin/activate  # Linux/Mac
# 或
venv\Scripts\activate  # Windows
```

3. 安装依赖：
```bash
pip install -r requirements.txt
```

4. 配置环境变量：
```bash
cp .env.example .env
# 编辑 .env，填入你的 API 密钥
```

### 启动服务

```bash
python main.py
```

服务将在 `http://localhost:8000` 启动。

访问 API 文档：`http://localhost:8000/docs`

## API 使用流程

### 步骤 1：注册医疗记录

```bash
curl -X POST "http://localhost:8000/api/simulation/register-record" \
  -H "Content-Type: application/json" \
  -d @- << 'EOF'
{
  "record_id": "patient_001",
  "patient_name": "张三",
  "patient_id": "P001",
  "age": 45,
  "gender": "男",
  "chief_complaint": "胸痛1周",
  "present_illness": "患者1周前开始出现左侧胸痛...",
  "key_symptoms": ["胸痛", "心悸", "气短"],
  "key_signs": ["心率102次/分", "血压140/90mmHg"],
  "lab_tests": {
    "troponin": "升高",
    "肌钙蛋白": "阳性"
  },
  "imaging_results": {
    "心电图": "ST段压低"
  },
  "primary_diagnosis": "急性心肌梗死",
  "differential_diagnoses": ["不稳定型心绞痛", "肺栓塞"],
  "treatment_plan": "急诊PCI治疗...",
  "medications": ["阿司匹林", "氯吡格雷"]
}
EOF
```

### 步骤 2：创建患者模拟

```bash
curl -X POST "http://localhost:8000/api/simulation/create" \
  -H "Content-Type: application/json" \
  -d @- << 'EOF'
{
  "medical_record_id": "patient_001",
  "personality": "anxious",
  "system_prompt": "你是一位45岁的男性患者，性格焦虑...",
  "trigger_keywords": {
    "symptoms": ["症状", "疼痛", "胸痛"],
    "signs": ["体征", "血压", "心率"],
    "lab_tests": ["检查", "化验"],
    "imaging": ["影像", "心电图"],
    "medications": ["治疗", "用药"]
  }
}
EOF
```

返回 `simulation_id`，用于后续操作。

### 步骤 3：与患者对话

```bash
curl -X POST "http://localhost:8000/api/conversation/message" \
  -H "Content-Type: application/json" \
  -d '{
    "simulation_id": "your_simulation_id",
    "message": "你好，请问你最近有什么不适吗？"
  }'
```

**响应示例**：
```json
{
  "patient_response": "医生，我最近感到胸部疼痛...",
  "triggered_keywords": ["symptoms"],
  "revealed_info": {
    "symptoms": true,
    "signs": false,
    "lab_tests": false,
    "imaging": false,
    "medications": false
  },
  "turn_id": 0
}
```

### 步骤 4：提交诊断

```bash
curl -X POST "http://localhost:8000/api/diagnosis/submit" \
  -H "Content-Type: application/json" \
  -d '{
    "simulation_id": "your_simulation_id",
    "diagnosis": "急性心肌梗死",
    "reasoning": "患者45岁，胸痛1周，心电图ST段压低，troponin升高..."
  }'
```

**响应示例**：
```json
{
  "overall_score": 85.5,
  "primary_diagnosis_match": 0.95,
  "differential_diagnosis_coverage": 0.75,
  "key_findings_identified": 0.85,
  "feedback": "总体得分: 85.5/100\n\n详细分析:\n- 主诊断匹配度: 95.0%\n- 鉴别诊断覆盖: 75.0%\n..."
}
```

## 配置说明

### Vector DB 选择

系统支持 Pinecone 和 Weaviate 两种向量数据库：

**Pinecone（推荐）**：
```bash
VECTOR_DB_TYPE=pinecone
PINECONE_API_KEY=your_key
PINECONE_ENVIRONMENT=us-east-1
```

**Weaviate**：
```bash
VECTOR_DB_TYPE=weaviate
WEAVIATE_URL=http://localhost:8080
WEAVIATE_API_KEY=your_key
```

### 患者性格类型

- `cooperative`：配合的患者，提供清晰完整的信息
- `anxious`：焦虑的患者，过度关注细节，重复提问
- `reluctant`：不情愿的患者，信息披露缓慢
- `dramatic`：戏剧化的患者，夸大症状表述

## 扩展方向

1. **多语言支持**：添加中英文切换
2. **患者库管理**：集成数据库管理大量病历
3. **性能优化**：使用 Redis 缓存对话历史
4. **案例导入**：支持从医学文献导入真实案例
5. **对标指标**：与其他医学教育平台的对比分析
6. **移动应用**：开发 iOS/Android 客户端

## 故障排除

### 连接错误
- 检查 OpenAI API 密钥是否有效
- 验证 Vector DB 连接配置

### 诊断评分不准确
- 调整 `TEMPERATURE` 参数控制 LLM 的创意性
- 优化关键词触发列表
- 检查医疗记录数据是否完整

## 贡献指南

欢迎提交 Issue 和 Pull Request！

## 许可证

MIT License