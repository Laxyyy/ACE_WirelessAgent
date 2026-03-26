## 修正后的 WirelessAgent 框架流程图

```mermaid
flowchart TD
    A[开始] --> B[Perception 感知模块]
    B --> B1[从 ray_tracing_results.csv 加载用户数据]
    B1 --> B2[获取用户 ID、位置、请求、CQI、地面真值]
    B2 --> C[Memory 记忆模块]
    
    C --> C1[初始化 state 状态]
    C1 --> C2[state 存储：history、memory、step_count、current_step、final_result]
    C2 --> D[Knowledge Base 知识库模块<br/>独立核心模块]
    
    D --> D1{KB/NKB 模式？}
    D1 -->|KB 模式 | D2[加载 Intent_Understand.txt 知识库]
    D2 --> D3[get_application_slice_type 函数<br/>基于关键词匹配推荐切片类型]
    D3 --> D4[存储结果到 state["memory"]["kb_recommended_slice"]]
    D1 -->|NKB 模式 | D5[跳过知识库处理]
    
    D4 --> E[Planning 规划模块<br/>LangGraph 四步工作流]
    D5 --> E
    
    E --> E1[Initialize 初始化节点]
    E1 --> E2[Understand Intent 意图理解节点]
    E2 --> E3[Allocate Slice Type 切片类型分配节点]
    E3 --> E4[Allocate Resources 资源分配节点]
    E4 --> E5[Evaluate Network 网络评估节点]
    
    E2 --> E2a[查询知识库获取推荐切片]
    E2 --> E2b[LLM 深度意图分析<br/>提取应用类型、网络需求、切片推荐]
    E2b --> E2c[比较 KB 与 LLM 推荐<br/>优先使用 KB 推荐]
    
    E3 --> E3a[检查工作负载平衡<br/>get_slice_utilization_rates]
    E3a --> E3b[利用率差异>20%?<br/>是则重新分配切片]
    
    E4 --> E4a[beamforming_tool<br/>LLM 推荐带宽 + 启发式规则]
    E4a --> E4b[calculate_rate_from_cqi<br/>香农公式：rate = bandwidth * log10(1 + SNR) * 10]
    E4b --> E4c[check_slice_capacity<br/>检查切片容量]
    E4c -->|容量不足 | E4d[check_and_adjust_capacity<br/>动态调整现有用户带宽]
    E4c -->|容量足够 | E4e[直接分配资源]
    
    E5 --> F[Action 行动模块]
    F --> F1[slice_allocation 工具<br/>执行切片分配]
    F1 --> F2[update_network_state<br/>更新全局网络状态]
    F2 --> F3[generate_concise_report<br/>生成详细报告]
    F3 --> F4[generate_user_allocation_table<br/>生成用户分配表格]
    F4 --> F5[export_results_to_csv<br/>导出结果到 CSV]
    F5 --> G[结束]
    
    subgraph "三大核心支撑"
        D[Knowledge Base 知识库模块]
        LLM[LLM 推理引擎]
        T[外部工具集]
    end
    
    subgraph "state 记忆模块<br/>仅存储中间结果"
        C1
        C2
    end
    
    subgraph "Planning 四步工作流"
        E1
        E2
        E3
        E4
        E5
    end
    
    subgraph "端到端流程"
        B
        C
        D
        E
        F
    end
```

## 修正后的 WirelessAgent 框架文字讲解

### 架构修正说明

经过代码验证，原流程图存在以下偏差点，现已修正：

---

### 偏差点 1 修正：知识库与记忆模块的边界

**原错误**：将知识库归为记忆模块的子功能

**修正后**：
- **Knowledge Base (知识库模块)**：独立核心模块，与 LLM、外部工具并列三大核心支撑
  - 实现：`load_knowledge_base()` 加载文件、`get_application_slice_type()` 检索匹配
  - 存储：独立的 `Knowledge_Base/Intent_Understand.txt` 文件
  - 功能：基于关键词匹配应用类型到切片类型 (eMBB/URLLC)
  
- **Memory (记忆模块)**：仅存储中间结果的容器
  - 实现：`state["memory"]` 字典，存储 `kb_recommended_slice`、`final_slice`、`beamforming_result` 等
  - 功能：缓存知识库检索结果、LLM 推理结果、工具调用结果
  - **不承载**知识库的核心能力（检索、匹配、推理）

**代码证据**：
```python
# 知识库是独立功能模块
def load_knowledge_base(file_path):
    """Load knowledge base from local file"""
    
def get_application_slice_type(application_type):
    """Determine the slice type from the knowledge base"""

# state["memory"] 仅存储结果
state["memory"]["kb_recommended_slice"] = kb_recommended_slice
state["memory"]["kb_slice_reasons"] = kb_reasons
```

---

### 偏差点 2 修正：带宽分配核心逻辑

**原错误**：描述为"执行波束赋形算法分配带宽"

**修正后**：
- **实际实现**：CQI 驱动的速率/吞吐量优化，无真实波束赋形算法
- **核心逻辑**：
  1. `beamforming_tool` 命名误导，实际是**LLM+ 启发式规则的带宽推荐**
  2. 带宽计算：LLM 分析请求 → 提取整数带宽值 → 启发式规则调整
  3. 速率计算：`calculate_rate_from_cqi()` 使用香农公式 `rate = bandwidth * log10(1 + SNR) * 10`
  4. 优化目标：吞吐量最大化，受限于总带宽约束

**代码证据**：
```python
# beamforming_tool 实际实现 - LLM 推荐带宽
bandwidth_prompt = f"""
Based on the following user request and network conditions, 
recommend an appropriate bandwidth allocation...
Please respond with a single integer number...
"""

# 速率计算基于香农公式，无波束赋形
def calculate_rate_from_cqi(bandwidth, cqi):
    snr = 10 ** (cqi / 10)
    rate = bandwidth * math.log10(1 + snr) * 10
    return round(rate, 2)
```

**修正说明**：虽然代码中有 `beamforming_tool` 命名，但实际实现是 LLM 驱动的带宽推荐器，核心算法是 CQI-MCS 映射的香农公式计算，无波束赋形相关代码。

---

### 偏差点 3 修正：规划模块定义

**原错误**：描述为"推理、检索、反思三大子模块"

**修正后**：
- **实际实现**：步骤化工作流（LangGraph 四节点）
  - `Initialize`：初始化状态
  - `Understand Intent`：意图理解（检索知识库+LLM 推理）
  - `Allocate Slice Type`：切片类型分配（含工作负载平衡检查）
  - `Allocate Resources`：资源分配（带宽推荐 + 容量检查 + 动态调整）
  - `Evaluate Network`：网络评估（结果输出）

- **缺失环节**：无反思 (Reflection) 子模块
  - 代码无结果回检、策略调整的反思逻辑
  - 仅通过 `Evaluate Network` 输出最终报告，无闭环优化

**代码证据**：
```python
# LangGraph 工作流节点定义
graph.add_node("initialize", initialize)
graph.add_node("understand_intent", understand_intent)
graph.add_node("allocate_slice_type", allocate_slice_type)
graph.add_node("allocate_resources", allocate_resources)
graph.add_node("evaluate_network", evaluate_network)

# 无反思节点
```

---

### 端到端流程修正

1. **Perception 感知**：从 CSV 加载用户数据（含 CQI 字段）
2. **Memory 记忆**：初始化 state 容器
3. **Knowledge Base 知识库**：独立模块，提供领域知识检索
4. **Planning 规划**：四步工作流
   - 意图理解：KB 检索 + LLM 推理，优先使用 KB 结果
   - 切片分配：工作负载平衡检查（利用率差异>20% 触发）
   - 资源分配：LLM 推荐带宽 → 香农公式计算速率 → 容量检查 → 动态调整
   - 网络评估：输出最终报告
5. **Action 行动**：执行分配、更新状态、导出 CSV

---

### 关键修正总结

| 偏差点 | 原描述 | 修正后 | 代码证据 |
|--------|--------|--------|----------|
| 知识库定位 | 记忆模块子功能 | 独立核心模块 | `load_knowledge_base()`独立函数 |
| 带宽分配 | 波束赋形算法 | LLM 推荐+CQI 香农公式 | `beamforming_tool` 实际实现 |
| 规划模块 | 推理/检索/反思子模块 | 四步工作流节点 | LangGraph 节点定义 |
| 反思环节 | 未说明缺失 | 明确标注代码未实现 | 无反思节点 |

这样的修正完全符合代码实现和论文定义，避免了过度解读和错误描述。