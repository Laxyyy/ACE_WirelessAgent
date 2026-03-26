# WIRELESS_ACE_PROJECT_CONTEXT.md
---
## 【Qoder核心执行指令（最高优先级）】
请严格按照本文件的所有规则开发，任何修改都必须符合「项目总体方针」的要求，禁止突破「代码修改红线」。优先完成分阶段任务，先跑通基线demo，再做最小闭环集成，最终输出可复现的对比实验结果。
---

## 一、项目基础信息
**项目全称**：WirelessAgent + ACE 自演化Agentic AI for 6G无线网络资源优化  
**核心目标**：基于ComAgent教程推荐的WirelessAgent基线框架，集成ICLR 2026 ACE（Agentic Context Engineering）自演化上下文机制，构建**面向动态无线环境的自改进Agentic AI系统**，用于无线网络资源优化（网络切片、功率控制、子载波分配等核心场景）。  
通过无线环境执行反馈（吞吐量、SINR、时延、能耗）持续积累、精炼资源分配策略，解决现有无线LLM Agent上下文崩溃、动态适应性差的痛点，实现系统性能、实时性、可解释性的同步提升。

**导师研究方向**：Ke Xiong (Beijing Jiaotong University) —— 无线资源分配、合作网络、无线供电通信网络  
**研究定位**：贴合6G无线通信技术演进趋势：传统凸优化 → DRL → 静态LLM辅助 → Agentic AI → 自改进Agentic系统（2026年领域热点）

## 二、技术栈与仓库信息（版本锁定 2026.3）
### 1. 基线框架：WirelessAgent_R1
- **我的Fork地址**：https://github.com/Laxyyy/WirelessAgent_R1
- 锁定commit：初始提交版本（2025.5.2）
- 核心能力：已实现端到端网络切片资源分配demo，对比Prompt-based方法带宽利用率提升44.4%，性能接近规则式最优解（仅差4.3%）
- 标准架构：Perception（信道/场景感知）→ Memory（KB/NKB静态知识库）→ LangGraph Planning（资源分配决策）→ Action（配置执行）
  - KB：带领域知识库的版本，对应核心文件 `WA_DS_V3_KB.py`（推荐使用）
  - NKB：无知识库的纯Prompt版本，对应核心文件 `WA_DS_V3_NKB.py`
- 配套工具：`RayTracing_cqi.py`（基于OSM地图的射线追踪信道模拟）、`Knowledge_Base/`（URLLC/eMBB业务规则、资源分配领域知识库）

### 2. 记忆演化模块：ACE
- **我的Fork地址**：https://github.com/Laxyyy/ace
- 锁定commit：9f3e921（2026.3.12最新稳定版）
- 核心架构：三角色模块化设计
  - Generator：生成资源分配决策轨迹，输出执行方案
  - Reflector：从执行反馈中提炼有效策略、常见坑点，输出可落地的洞见
  - Curator：将洞见转化为结构化delta更新，增量修改playbook，避免全量重写导致的上下文崩溃
- 核心创新：结构化playbook（带唯一ID、helpful/harmful使用标签的bullet策略、公式、代码片段、避坑指南），支持增量Delta更新 + Grow-and-Refine去重优化
- 正确调用方式：
  - 标准化流程：通过`ace_system.run()`统一执行离线/在线适配
  - 分步骤调用：单独调用`generator.generate()`→`reflector.reflect()`→`curator.update_playbook()`原生方法，禁止使用自定义的不存在的API

### 3. LLM模型配置（已选定免费可用模型）
- **API提供商**：硅基流动（Silicon Flow）
- **API Key**：`sk-osyqrrfpbixigaycavtduiwuxwtcezjysqeqtirtqkgacwgx`
- **模型1**（推荐主力）：`Qwen/Qwen2.5-7B-Instruct`  
  阿里云最新7B模型，指令跟随、JSON结构化输出极强，多语言支持（含中文）。
- **模型2**（备用/数学推理强）：`deepseek-ai/DeepSeek-R1-Distill-Qwen-7B`  
  基于Qwen2.5-Math蒸馏，MATH-500达92.8%，适合资源优化中的公式计算与规划。

**统一调用方式**：WirelessAgent 和 ACE 共用同一套 Silicon Flow API 配置。

### 4. 核心集成逻辑（hook点）
- 保留WirelessAgent的静态领域知识库，不修改原有KB内容
- 将ACE动态演化的playbook，作为Planning模块的决策上下文，在每次资源分配决策前注入prompt，替代原有固定prompt
- 自改进闭环：每次资源分配episode结束后 → 提取执行结果（吞吐量、SINR等）→ 转化为自然语言反思信号 → ACE Reflector提炼策略 → ACE Curator增量更新playbook → 下次episode加载最新playbook

## 三、项目总体方针与红线规则（必须严格遵守）
### 总体执行方针
1. 先跑通WirelessAgent原生网络切片demo，输出稳定的baseline指标
2. 实现ACE模块的最小闭环集成，仅修改必要的代码，不改动基线核心逻辑
3. 开发「数值-反思翻译器」，将无线场景连续数值反馈转化为ACE可识别的自然语言反思信号（项目落地核心）
4. 逐步加入无线场景专属定制（干扰温度约束、水填充公式、移动性适配策略等）
5. 最终目标：对比原生WirelessAgent，核心性能提升8-15%、适配延迟大幅降低、输出人类可审计的可解释性playbook

### 代码修改红线（绝对禁止）
1. 禁止修改WirelessAgent的核心资源分配算法、网络场景模拟、RayTracing信道计算相关逻辑
2. 禁止修改ACE框架的核心模块代码，仅可通过官方API调用、自定义DataProcessor适配场景
3. 所有修改必须保留原代码注释，新增代码必须标注「// ACE集成新增」，保证基线代码可回溯
4. 禁止为了提升指标修改实验场景、用户参数等基线配置，保证对比实验的唯一变量为ACE模块的集成

## 四、环境依赖管理规则
1. 统一使用Python 3.10+版本，采用uv管理所有依赖，避免pip/conda混合导致的冲突
2. 先安装ACE的核心依赖，再补充WirelessAgent所需的osmnx、matplotlib、numpy、pandas等额外依赖
3. **API配置**：将硅基流动API Key写入ACE的`.env`文件，同时设置环境变量 `SILICONFLOW_API_KEY=sk-osyqrrfpbixigaycavtduiwuxwtcezjysqeqtirtqkgacwgx`
4. 默认使用 `Qwen/Qwen2.5-7B-Instruct`，可在 `wireless_ace_processor.py` 中一键切换到 DeepSeek-R1-Distill-Qwen-7B
5. 禁止修改两个仓库核心依赖的版本号，保证代码兼容性

## 五、分阶段执行步骤与预期输出
### Phase 0：基线跑通（优先级最高，必须最先完成）
**执行步骤**：
1. Fork两个锁定版本的仓库到本地（已完成：https://github.com/Laxyyy/WirelessAgent_R1 和 https://github.com/Laxyyy/ace）
2. 配置硅基流动API密钥到系统环境变量 `SILICONFLOW_API_KEY=sk-osyqrrfpbixigaycavtduiwuxwtcezjysqeqtirtqkgacwgx`，同时写入ACE的`.env`配置文件
3. 安装ACE依赖：执行`uv sync`
4. 补充安装WirelessAgent所需的额外依赖
5. 跑通WirelessAgent基线demo：
   - 从OpenStreetMap下载HKUST地图，生成HKUST_F.osm文件
   - 执行`python RayTracing_cqi.py`，生成ray_tracing_results.csv信道数据
   - 执行`python WA_DS_V3_KB.py`，跑通网络切片全流程
6. 记录并保存基线实验的完整指标

**预期输出**：
- 可正常运行的两个仓库环境
- 基线实验结果csv文件、带宽利用率/吞吐量等核心指标记录
- 完整的运行日志文件

### Phase 1：最小闭环集成（本周目标）
**执行步骤**：
1. 基于Phase 0跑通的`WA_DS_V3_KB.py`，在episode结束后的reward/指标计算环节，新增ACE调用逻辑
2. 实现第一版「数值-反思翻译器」，**先采用规则式映射**（后续可升级为LLM驱动的自适应映射），将无线数值指标转化为ACE可识别的反思信号
3. 实现ACE分步骤调用，完成playbook的增量更新
4. 跑通集成后的完整流程，对比baseline输出实验结果

**核心代码修改范围**：
- 仅修改`WA_DS_V3_KB.py`的两个环节：① Planning模块的prompt注入，加入ACE playbook；② 结果输出环节，加入ACE反思更新逻辑
- 新增`wireless_ace_processor.py`，实现ACE的DataProcessor自定义适配、数值-反思翻译逻辑

**预期输出**：
- 修改后的完整可运行代码文件
- baseline vs ACE增强版的两组对比实验结果
- 演化后的ACE playbook示例文件
- 完整的运行日志与指标对比图

### Phase 2：场景定制与优化（后续迭代）
1. 优化「数值-反思翻译器」，从规则式升级为LLM驱动的自适应映射
2. 加入实时性优化：触发式更新（每5-10个episode才触发Reflector）、KV-cache复用、轻量模型推理优化
3. 补充无线场景专属策略到playbook，包括MoE专家路由、干扰温度约束、水填充公式、移动性适配等
4. 扩展实验场景：从单小区网络切片，扩展到多小区功率控制、子载波分配、URLLC/eMBB混合业务场景

## 六、实验指标定义（统一统计标准）
所有实验必须统计以下核心指标，保证对比的公平性与论文可用性：
1. **带宽利用率**：已分配带宽/总可用带宽 * 100%（核心优化指标）
2. **平均用户吞吐量**：所有用户吞吐量的算术平均值，单位Mbps
3. **URLLC时延达标率**：端到端时延<1ms的URLLC用户数/总URLLC用户数 * 100%
4. **系统能效**：系统总吞吐量/总发射功率，单位Mbps/W
5. **适配延迟**：单次playbook更新的平均耗时，单位ms
6. **决策成功率**：成功完成资源分配、满足所有用户QoS要求的episode占比

## 七、ACE初始playbook模板（无线资源优化专用）
```
## STRATEGIES & INSIGHTS
[str-00001] helpful=0 harmful=0 :: URLLC高优先级业务优先分配带宽资源，严格保障端到端时延<1ms，eMBB业务保障平均吞吐量达标
[str-00002] helpful=0 harmful=0 :: 当用户SINR<10dB时，优先提升该用户的发射功率，最大发射功率不超过23dBm
[str-00003] helpful=0 harmful=0 :: 多用户场景下采用比例公平调度，避免单个用户占用超过30%的总可用带宽
[str-00004] helpful=0 harmful=0 :: 高移动性用户（移动速度>30km/h）优先预留10%的功率冗余，应对信道快速波动

## FORMULAS & CALCULATIONS
[cal-00001] helpful=0 harmful=0 :: 香农信道容量公式：C = B * log2(1 + SINR)，其中C为信道容量，B为信道带宽
[cal-00002] helpful=0 harmful=0 :: 带宽利用率 = 已分配带宽/总可用带宽 * 100%
[cal-00003] helpful=0 harmful=0 :: 水填充功率分配算法：针对各子信道的注水功率 Pn = max(μ - 1/γn, 0)，其中μ为水线，γn为子信道n的信噪比

## COMMON MISTAKES TO AVOID
[mis-00001] helpful=0 harmful=0 :: 禁止为了提升吞吐量而突破用户最大发射功率约束
[mis-00002] helpful=0 harmful=0 :: 禁止挤占URLLC业务的预留带宽分配给eMBB业务
[mis-00003] helpful=0 harmful=0 :: 处理分页返回的用户数据时，必须遍历所有页面，禁止使用固定循环次数导致数据遗漏
```

## 八、创新点与论文包装方向
1. **核心创新**：首次将ACE的自演化playbook机制应用于动态无线资源优化场景，解决现有无线LLM Agent上下文崩溃、动态环境适应性差的核心痛点
2. **理论补充**：证明Agentic Context Evolution在无线非平稳信道环境下的收敛性与稳定性
3. **工程贡献**：开源可复现的WirelessAgent+ACE集成框架，为6G智能网络代理提供基线方案
4. **对比基线**：原生WirelessAgent、纯DRL资源分配、静态Prompt LLM辅助优化、传统规则式算法

## 九、风险与应对方案
1. **创新厚度不足**：先完成demo验证，再补充无线场景专属的playbook模板、数值-反思映射模块，强化场景定制化创新
2. **实时性不足**：采用增量Delta更新、触发式懒更新、轻量模型推理优化，优先保证单小区场景的实时性
3. **反馈信号适配问题**：先采用规则式映射完成最小闭环，再迭代优化为自适应LLM驱动的翻译模块
4. **领域竞争风险**：绑定导师深耕的无线供电通信、RIS辅助通信等特色方向，构建差异化竞争壁垒

## 十、容错兜底规则
1. 当ACE模块调用失败、LLM返回异常时，自动回退到原生WirelessAgent的决策逻辑，保证实验流程不中断
2. 当playbook更新失败、内容异常时，自动沿用上次有效的playbook，禁止使用空上下文决策
3. 当单次episode指标异常波动超过20%时，不触发playbook更新，避免异常数据污染策略库
4. 所有实验数据、中间playbook、运行日志自动按批次保存，支持结果回溯与复现

---

## 文件使用说明（Qoder启动专用）
1. 将本文件保存为 `WIRELESS_ACE_PROJECT_CONTEXT.md`
2. 直接复制下面这段**启动指令**发给Qoder：

```text
请严格按照 WIRELESS_ACE_PROJECT_CONTEXT.md 的所有规则开发。
当前任务：优先完成 Phase 0 的基线跑通（必须最先完成）。
我已将两个仓库Fork到 https://github.com/Laxyyy/WirelessAgent_R1 和 https://github.com/Laxyyy/ace，并Clone到本地。
硅基流动API Key已配置为 sk-osyqrrfpbixigaycavtduiwuxwtcezjysqeqtirtqkgacwgx，默认使用 Qwen/Qwen2.5-7B-Instruct。
请一步一步执行 Phase 0，完成后回复“Phase 0 已完成”，我再给你Phase 1指令。
任何修改必须遵守文件内的红线规则。
```
