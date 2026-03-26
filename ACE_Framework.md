

## ACE 框架流程图（基于代码验证）

```mermaid
flowchart TD
    A[开始] --> B[ACE 系统初始化]
    B --> B1[初始化三个核心 Agent<br/>Generator、Reflector、Curator]
    B1 --> B2[初始化 Playbook<br/>空 playbook 或加载初始 playbook]
    B2 --> B3[配置运行模式<br/>offline/online/eval_only]
    
    B3 --> C{运行模式？}
    
    C -->|Offline 模式 | C1[离线训练流程]
    C -->|Online 模式 | C2[在线训练测试流程]
    C -->|Eval Only 模式 | C3[仅评估流程]
    
    subgraph OfflineMode [离线训练模式]
        C1 --> O1[加载训练集和验证集]
        O1 --> O2[多 Epoch 迭代训练]
        O2 --> O3[每个样本执行单样本训练流程]
        O3 --> O4[定期评估验证集]
        O4 --> O5[保存最佳 playbook]
    end
    
    subgraph OnlineMode [在线训练测试模式]
        C2 --> N1[加载测试样本]
        N1 --> N2[逐窗口处理]
        N2 --> N3[先测试当前窗口<br/>用当前 playbook]
        N3 --> N4[再训练当前窗口<br/>更新 playbook]
        N4 --> N5[累积测试结果]
    end
    
    subgraph SingleSampleTraining [单样本训练流程<br/>_train_single_sample]
        O3 --> S1[STEP 1: Generator 初始生成]
        S1 --> S2[检查答案正确性]
        
        S2 -->|答案错误 | S3[STEP 2: 多轮反思迭代]
        S3 --> S3a[Reflector 反思错误<br/>最多 max_num_rounds 轮]
        S3a --> S3b[Generator 基于反思重新生成]
        S3b --> S3c[检查是否纠正]
        S3c -->|仍未纠正 | S3a
        
        S2 -->|答案正确 | S4[STEP 2: 反思标记有用条目]
        S4 --> S4a[Reflector 标记 helpful 条目]
        
        S3c -->|已纠正 | S5[STEP 3: Curator 定期更新]
        S4a --> S5
        
        S5 --> S5a[检查 curator_frequency]
        S5a -->|达到更新频率 | S5b[Curator 执行增量更新<br/>ADD/UPDATE/MERGE/DELETE]
        S5b --> S5c[BulletpointAnalyzer 去重优化<br/>grow-and-refine]
        S5a -->|未达到频率 | S6[STEP 4: Post-Curator 生成]
        S5c --> S6
        
        S6 --> S6a[Generator 使用更新后的<br/>playbook 重新生成]
        S6a --> S6b[记录 pre-train 和 post-train 结果]
    end
    
    subgraph EvalOnlyMode [仅评估模式]
        C3 --> E1[加载测试样本]
        E1 --> E2[使用当前 playbook 评估]
        E2 --> E3[输出测试结果]
    end
    
    O5 --> F[结束]
    N5 --> F
    E3 --> F
    
    subgraph GeneratorDetail [Generator 核心逻辑]
        S1 -.-> G1[输入：question、playbook、context、reflection]
        G1 -.-> G2[加载 GENERATOR_PROMPT]
        G2 -.-> G3[LLM 调用生成回答]
        G3 -.-> G4[提取 bullet_ids<br/>正则匹配 [xxx-00001]]
        G4 -.-> G5[输出：full_response、bullet_ids、call_info]
    end
    
    subgraph ReflectorDetail [Reflector 核心逻辑]
        S3a -.-> R1[输入：question、reasoning_trace、<br/>predicted_answer、ground_truth、<br/>bullets_used]
        R1 -.-> R2[选择 REFLECTOR_PROMPT<br/>有/无 GT 版本]
        R2 -.-> R3[LLM 调用反思]
        R3 -.-> R4[提取 bullet_tags<br/>helpful/harmful/neutral]
        R4 -.-> R5[输出：reflection_content、<br/>bullet_tags、call_info]
    end
    
    subgraph CuratorDetail [Curator 核心逻辑]
        S5b -.-> C4[输入：current_playbook、<br/>recent_reflection、<br/>question_context、<br/>token_budget]
        C4 -.-> C5[选择 CURATOR_PROMPT<br/>有/无 GT 版本]
        C5 -.-> C6[LLM 调用生成 operations JSON]
        C6 -.-> C7[验证 JSON schema<br/>operations 数组]
        C7 -.-> C8[apply_curator_operations<br/>执行 ADD/UPDATE/MERGE/DELETE]
        C8 -.-> C9[更新 next_global_id]
        C9 -.-> C10[输出：updated_playbook、<br/>next_global_id、operations]
    end
    
    subgraph PlaybookStructure [Playbook 结构]
        P1[## STRATEGIES & INSIGHTS<br/>策略与洞察]
        P2[## FORMULAS & CALCULATIONS<br/>公式与计算]
        P3[## CODE SNIPPETS & TEMPLATES<br/>代码片段与模板]
        P4[## COMMON MISTAKES TO AVOID<br/>常见错误与避坑]
        P5[## PROBLEM-SOLVING HEURISTICS<br/>问题解决启发式]
        P6[## CONTEXT CLUES & INDICATORS<br/>上下文线索与指示]
        P7[## OTHERS<br/>其他]
        
        P1 --- P8[条目格式：<br/>[str-00001] helpful=5 harmful=0 :: 内容]
    end
    
    subgraph KeyMechanisms [核心机制]
        K1[增量更新机制<br/>非整体重写，避免上下文坍塌]
        K2[Grow-and-Refine<br/>语义去重 + 剪枝优化]
        K3[条目标签追踪<br/>helpful/harmful 计数]
        K4[多轮反思迭代<br/>最多 5 轮，确保洞察精准]
        K5[定期 Curator 更新<br/>curator_frequency 控制频率]
    end
    
    style GeneratorDetail fill:#e3f2fd,stroke:#1565c0
    style ReflectorDetail fill:#f3e5f5,stroke:#7b1fa2
    style CuratorDetail fill:#e8f5e9,stroke:#2e7d32
    style PlaybookStructure fill:#fff9c4,stroke:#f9a825
    style KeyMechanisms fill:#ffebee,stroke:#c62828
    style SingleSampleTraining fill:#f5f5f5,stroke:#616161
```

## ACE 框架文字讲解

### 架构分层与核心组件

ACE 框架采用**三角色模块化架构**，替代传统"单个 LLM 一次性完成全流程"的模式：

---

### 1. Generator（生成器）- 执行入口

**核心职责**：基于 playbook 生成任务执行轨迹

**代码实现**（`ace/ace/core/generator.py`）：
```python
def generate(self, question, playbook, context, reflection, use_json_mode, call_id, log_dir):
    # 1. 格式化 prompt
    prompt = GENERATOR_PROMPT.format(playbook, reflection, question, context)
    
    # 2. LLM 调用生成回答
    response, call_info = timed_llm_call(...)
    
    # 3. 提取 bullet IDs（正则匹配 [xxx-00001]）
    bullet_ids = self._extract_bullet_ids(response, use_json_mode)
    
    # 4. 返回：完整回答、使用的 bullet IDs、调用信息
    return response, bullet_ids, call_info
```

**关键特性**：
- 从 playbook 检索相关条目（通过 bullet IDs 标记）
- 支持 reflection 输入（用于反思后重新生成）
- 预标记使用的 playbook 条目（helpful/harmful/neutral）

---

### 2. Reflector（反思器）- 诊断大脑

**核心职责**：从执行反馈中提炼可复用的洞察

**代码实现**（`ace/ace/core/reflector.py`）：
```python
def reflect(self, question, reasoning_trace, predicted_answer, ground_truth, 
            environment_feedback, bullets_used, use_ground_truth, use_json_mode, call_id, log_dir):
    # 1. 选择 prompt（有/无 GT 版本）
    if use_ground_truth and ground_truth:
        prompt = REFLECTOR_PROMPT.format(...)
    else:
        prompt = REFLECTOR_PROMPT_NO_GT.format(...)
    
    # 2. LLM 调用反思
    response, call_info = timed_llm_call(...)
    
    # 3. 提取 bullet_tags（helpful/harmful/neutral）
    bullet_tags = self._extract_bullet_tags(response, use_json_mode)
    
    # 4. 返回：反思内容、条目标签、调用信息
    return response, bullet_tags, call_info
```

**关键特性**：
- 支持多轮迭代精炼（最多`max_num_rounds=3`轮）
- 从成功/失败案例中提炼通用策略
- 校验 Generator 预标记的条目有用性
- 输出结构化反思：错误定位、根因分析、正确方案、可复用洞察

---

### 3. Curator（策展器）- 记忆管理员

**核心职责**：将反思洞察转换为增量更新，避免上下文坍塌

**代码实现**（`ace/ace/core/curator.py`）：
```python
def curate(self, current_playbook, recent_reflection, question_context, 
           current_step, total_samples, token_budget, playbook_stats, 
           use_ground_truth, use_json_mode, call_id, log_dir, next_global_id):
    # 1. 选择 prompt（有/无 GT 版本）
    prompt = CURATOR_PROMPT.format(...)
    
    # 2. LLM 调用生成 operations JSON
    response, call_info = timed_llm_call(...)
    
    # 3. 验证 JSON schema
    operations_info = self._extract_and_validate_operations(response)
    operations = operations_info["operations"]
    
    # 4. 执行增量更新操作
    updated_playbook, next_global_id = apply_curator_operations(
        current_playbook, operations, next_global_id
    )
    
    # 5. 返回：更新后的 playbook、下一个全局 ID、操作列表
    return updated_playbook, next_global_id, operations, call_info
```

**关键特性**：
- **增量更新**：非整体重写，避免上下文坍塌
- **确定性操作**：ADD/UPDATE/MERGE/DELETE（目前仅 ADD 完全实现）
- **Grow-and-Refine**：BulletpointAnalyzer 语义去重 + 剪枝优化
- **元数据追踪**：每个条目有唯一 ID、helpful/harmful 计数

---

### 4. Playbook（结构化操作手册）- 核心记忆体

**数据结构**（`playbook_utils.py`）：
```
## STRATEGIES & INSIGHTS
[str-00001] helpful=5 harmful=0 :: URLLC 高优先级业务优先分配带宽资源

## FORMULAS & CALCULATIONS
[cal-00001] helpful=3 harmful=0 :: 香农信道容量公式：C = B * log2(1 + SINR)

## COMMON MISTAKES TO AVOID
[mis-00001] helpful=2 harmful=1 :: 禁止为了提升吞吐量而突破用户最大发射功率约束
```

**条目格式**：`[ID] helpful=X harmful=Y :: 内容`

**核心特性**：
- 条目化存储（支持细粒度检索）
- 增量更新（支持并行合并）
- 元数据追踪（helpful/harmful计数）
- 避免整体重写（100%保留历史有效知识）

---

### 5. 运行模式

#### Offline 模式（离线训练）
- **流程**：批量处理训练集 → 多 Epoch 迭代 → 定期评估验证集 → 保存最佳 playbook
- **适用场景**：有标注数据、任务固定、需提前优化通用上下文

#### Online 模式（在线训练测试）
- **流程**：逐窗口处理 → 先测试后训练 → 实时更新 playbook → 累积测试结果
- **适用场景**：无标注数据、任务动态变化、需实时自提升

#### Eval Only 模式（仅评估）
- **流程**：使用现有 playbook 评估测试集
- **适用场景**：推理阶段、对比实验

---

### 6. 核心机制

#### 增量更新机制
- **传统方法**：整体重写 playbook → 上下文坍塌（丢失历史知识）
- **ACE 方法**：条目化增量更新 → 100% 保留历史有效知识

#### Grow-and-Refine 优化
- **去重**：BulletpointAnalyzer 语义嵌入对比，合并高度重复条目
- **剪枝**：清理无效、冗余条目，控制 playbook 规模
- **整理**：按模块分类，保证结构化

#### 条目标签追踪
- 每次 Reflector 反思后更新 helpful/harmful 计数
- 长期 harmful 的条目可被标记删除
- 支持多轮迭代强化有效策略

#### 多轮反思迭代
- 最多`max_num_rounds=5`轮
- 确保洞察精准、根因定位准确
- 避免泛泛的模糊结论

#### 定期 Curator 更新
- `curator_frequency`控制更新频率（如每 1 个样本更新一次）
- 平衡更新及时性与计算开销

---

### 7. 全链路数据流转

1. **输入**：Question + Context Playbook → Generator
2. **生成**：Task Trajectory + Bullet IDs → Execution Environment
3. **执行**：Execution Feedback + Ground Truth → Reflector
4. **反思**：Reflection Insight + Bullet Tags → Curator
5. **策展**：Delta Context Items → 更新 Playbook
6. **闭环**：更新后的 Playbook → 下一轮输入

形成完整的自提升迭代闭环。