# Agentic AI in 5G–6G Networks
**Personal Research Notes — AI-RAN · O-RAN SMO · Agentic AI**

> I work in 5G/6G. This repository is my personal knowledge base as I learn the AI side of the RAN stack. Everything here is written from papers, specs, and whitepapers in my own words so I can actually understand the concepts, not just reference them.

---

## Table of Contents

- [How to Read This Repo](#how-to-read-this-repo)
- [Part 1 — What is Agentic AI](#part-1--what-is-agentic-ai)
- [Part 2 — O-RAN Architecture and SMO](#part-2--o-ran-architecture-and-smo)
- [Part 3 — AI-RAN](#part-3--ai-ran)
- [Part 4 — Agentic AI Applied to RAN](#part-4--agentic-ai-applied-to-ran)
- [Part 5 — 6G and AI-Native Networks](#part-5--6g-and-ai-native-networks)
- [Part 6 — Agentic Design Patterns](#part-6--agentic-design-patterns)
- [Part 7 — Governance Safety and Ethics](#part-7--governance-safety-and-ethics)
- [Papers Reading List](#papers-reading-list)
- [Reading Order](#reading-order)
- [Glossary](#glossary)

---

## How to Read This Repo

This is not a course. It is my personal study material — concepts that took me longer to understand are explained in more detail, concepts that came quickly are shorter. If you already have a strong O-RAN background, go to Part 1 first to build the agentic AI foundation, then come back to Part 2 to see where agents fit in the architecture. If you already know agentic AI, go directly to Part 4 where both streams converge.

---

## Part 1 — What is Agentic AI

### The Difference Between Generative AI and Agentic AI

When LLMs first went mainstream, everyone saw the same pattern — you type something, the model generates a response. That is a one-shot input-output mapping. The model has no goal, no plan, no memory of what happened before, and no feedback loop telling it whether its output was correct. That is generative AI.

Agentic AI is fundamentally different. The ReAct paper (Yao et al., arXiv:2210.03629) was the first to formally show that LLMs are not limited to generating text — they can generate both verbal reasoning traces and task-specific actions in an interleaved manner. As the paper describes it, reasoning traces help the model induce, track, and update action plans as well as handle exceptions, while actions allow it to interface with external sources such as knowledge bases or environments to gather additional information. This means the model thinks in one step and does something in the next, then observes the result and thinks again. The loop continues until the goal is reached.

The Agentic AI-RAN paper (arXiv:2602.24115) defines this in the O-RAN context: agentic AI systems with explicit planning, tool use, memory, and self-management offer a natural way to structure long-lived control loops. Long-lived means this is not a one-time decision — the agent continuously monitors the network, makes changes, and learns from its own performance over time.

### The ReAct Loop — Walking Through a Complete Cycle

The ReAct loop is the execution engine of every LLM-based agentic system. It is worth understanding exactly how one complete cycle runs, because every agentic RAN paper is a variant of this loop applied to a specific network problem.

The loop has four alternating phases: Observe, Thought, Action, Observation. The agent first receives an observation about the current state of the world. It then generates a Thought — a verbal reasoning trace explaining what the observation means and what should be done next. Based on that Thought, it generates an Action — a specific tool call with parameters. The tool executes and returns an Observation. That Observation becomes the input to the next Thought. This continues until the agent generates a Final Answer.

What makes this powerful is the Thought step. Before acting, the agent writes out its reasoning in natural language. This forces it to work through the logic explicitly, which means it can catch mistakes in its own reasoning before executing anything. The ReAct paper showed this is why ReAct overcomes issues of hallucination and error propagation prevalent in chain-of-thought reasoning alone — by interacting with real environments, the agent corrects itself through observations rather than hallucinating a path to a conclusion.

In practice, ReAct is implemented as a structured prompt template. The framework parses the Action section from the LLM output, calls the actual tool or API, appends the result as an Observation to the conversation context, and calls the LLM again. The LLM now sees the full history — original goal, all past Thoughts, all past Actions, all past Observations — and generates the next Thought. This continues until the LLM outputs a stop condition.

### Planning — Breaking a Goal into Steps

The planning module breaks a high-level goal into a concrete sequence of steps. This is fundamentally different from an RL xApp, which maps the current state directly to an action through a learned policy. That mapping was built during training and fails in novel situations the model never encountered. A planning-based agent reasons from first principles, constructs steps, and then executes — which is why it can handle situations it has never seen before.

The Agentic AI-RAN paper (arXiv:2602.24115) formalizes this as the Plan-Act-Observe-Reflect primitive. At each decision epoch the agent receives a context made up of KPM/KQI snapshots, topology, and slice mix, along with a goal that encodes objectives, constraints including isolation and fairness, and budgets for latency, CPU, and E2 bandwidth. The agent then selects a short sequence of O-RAN skills — such as PRB reallocation, power capping, or handover pinning — and executes them. This is the Plan phase.

### Memory — How the Agent Remembers

Without memory, an agentic system starts fresh every time. It has no record of what worked yesterday, what failed last week, or what patterns repeat every Friday evening. The RAN Cortex paper (arXiv:2505.07842) specifically identifies this as the core problem: these agents remain fundamentally stateless, treating each decision as isolated, lacking any persistent memory of prior events or outcomes. This reactive behavior constrains optimization, especially in environments where network dynamics exhibit episodic or recurring patterns.

The CoALA framework (arXiv:2309.02427, Princeton) formally categorizes memory in AI agents into four types drawn from cognitive science. In-context memory is whatever is inside the LLM context window during the current session. Episodic memory holds records of specific past events. Semantic memory holds general factual knowledge. Procedural memory holds rules and skills — knowledge about how things are done.

RAN Cortex implements these memory types for O-RAN through a specific four-element architecture: a context encoder that transforms network state into high-dimensional embeddings, a vector-based memory store of past network episodes, a recall engine that retrieves semantically similar past situations, and a policy interface that supplies historical context to AI agents in real time or near-real time. When a new network event arrives, the agent first retrieves similar past episodes from memory and incorporates them into the current decision. This is how the agent improves over time without retraining.

### Tool Use — How the Agent Interacts with the Real World

Tools are what connect an LLM-based agent to the real world. Without tools, the agent only generates text. With tools, it can actually do things. The Agentic AI-RAN paper (arXiv:2602.24115) introduces this as the Skills as Tool-Use primitive. In O-RAN, a skill is described as a thin, verifiable wrapper around a controllable primitive that the agent invokes as a tool. The verifiable part matters — the operator must be able to know exactly what a skill does and what its effect is, and this must be automatically auditable.

The paper organizes the task landscape into three clusters where these skills operate: network slice life-cycle management, radio resource management closed loops, and cross-cutting security, privacy, and compliance. Each cluster has different skills that use different O-RAN interfaces — a slice lifecycle skill uses O1 and O2, an RRM skill uses E2SM-RC, a monitoring skill uses E2SM-KPM. The abstraction is what makes this work: the high-level planning agent does not need to know the E2AP protocol details. It calls the skill by name with semantic parameters, and the skill handles the protocol translation.

### Reflection — How the Agent Catches Its Own Mistakes

The ReAct paper (arXiv:2210.03629) showed that ReAct generates human-like task-solving trajectories that are more interpretable than baselines without reasoning traces. This happens because when the agent generates a Thought before acting, it can catch mistakes in its own reasoning before it acts. Reflection goes one step further — after taking an action, the agent observes what happened and evaluates whether it matches the expected outcome.

The Agentic AI-RAN paper (arXiv:2602.24115) formalizes this as the Observe-Reflect phase. Memory spans short-term caches at the Near-RT level, episodic logs per decision, and long-term knowledge curated at the higher layers. Evidence is generated by design at each commit — decision-level records link goals, compact context, selected actions, and observed outcomes, supporting audit and regulatory reporting without exposing raw subscriber data. This is the key design insight: reflection is not just an internal reasoning process but a documented audit trail that satisfies regulatory requirements.

### Multi-Agent Systems — Why One Agent Is Not Enough

The Wireless Large AI Model paper (arXiv:2504.14653) describes agentic AI-RAN as enabling systems to perceive, reason, decide, and act autonomously. The challenge is that optimizing multiple objectives simultaneously — resource allocation, service assurance, energy efficiency — is too complex for a single agent operating alone.

The Agentic AI for Intent-driven Optimization paper (arXiv:2602.22539) implements a concrete multi-agent structure in cell-free O-RAN. A supervisor agent translates operator intents into an optimization objective and minimum rate requirements. A user weighting agent retrieves relevant prior experience from a memory module to determine user priority weights for precoding. If the intent includes an energy-saving objective, an O-RU management agent is activated to determine the set of active O-RUs using a deep reinforcement learning algorithm. A monitoring agent continuously measures user data rates and coordinates with the other agents to guarantee minimum rate requirements are satisfied. The paper reports that this framework reduces the number of active O-RUs by a factor of four while maintaining the minimum rate requirements. To enhance scalability, the paper adopts a parameter-efficient fine-tuning method that enables the same underlying LLM to be used across different agents — a practical solution to the cost problem of running multiple large models simultaneously.

---

## Part 2 — O-RAN Architecture and SMO

### Why O-RAN Exists

Traditional RAN was a closed ecosystem. You bought a complete base station from a single vendor — radio hardware, baseband processing software, management interface — all proprietary, all from the same company, communicating through proprietary protocols. You could not mix components from different vendors. You could not write your own optimization algorithm and run it inside the RAN. All innovation came from a handful of major vendors.

Polese et al. (arXiv:2202.01032) — the most widely cited O-RAN survey — describes this problem and O-RAN's solution: define open interfaces between all components of the RAN stack so different vendors can interoperate, and define a RAN Intelligent Controller platform where third parties can deploy their own AI/ML optimization applications.

### The O-RAN Disaggregation

The traditional monolithic base station is split into three separate software-defined network functions.

The O-RU (Open Radio Unit) is the physical radio hardware — antenna, RF electronics, and the very bottom of L1. It connects to the O-DU via the fronthaul interface using eCPRI (enhanced Common Public Radio Interface). The fronthaul has extremely tight latency requirements — in C-RAN deployments, under 100 microseconds one way.

The O-DU (Open Distributed Unit) handles the remaining L1 (channel coding — LDPC encode/decode, rate matching), L2 MAC (scheduling, HARQ), and L2 RLC. The O-DU communicates with the O-CU via the F1 interface — F1-C for the control plane and F1-U for the user plane.

The O-CU (Open Centralized Unit) handles the upper protocol layers: RRC (Radio Resource Control — connection management, measurement reporting, handover decisions), SDAP, and PDCP. The O-CU is split into O-CU-CP (Control Plane) and O-CU-UP (User Plane) connected via the E1 interface. The O-CU connects to the 5G Core via N2 and N3.

### SMO — Architecture and Internal Structure

The SMO (Service Management and Orchestration) is the top layer of the O-RAN management hierarchy and the most important layer for understanding where agentic AI lives at the highest timescale. It is not simply a management interface — it is a full platform that contains the Non-RT RIC, manages the O-Cloud infrastructure, and provides the data foundation on which all AI/ML in O-RAN is built.

The SMO manages O-RAN network functions through two primary interfaces toward the network. The O1 interface connects the SMO to the O-DU, O-CU, and O-RU. Through O1, the SMO configures network functions by pushing YANG data models via NETCONF, collects performance management data in the form of PM files at defined collection intervals, receives fault management alarms in real time, and pushes software lifecycle management operations. O1 is essentially the configure-and-observe interface — the SMO tells the network functions how to behave and collects data about how they are actually behaving. The O2 interface connects the SMO to the O-Cloud. The O-Cloud is the cloud infrastructure — servers, storage, networking — on which O-RAN network functions run as containerized workloads, typically managed through Kubernetes. Through O2, the SMO deploys new network function instances, scales them up or down based on traffic demand, manages the hardware resource pools, and handles the full lifecycle of containerized O-RAN functions from instantiation to termination.

### Non-RT RIC — The Intelligence Layer Inside the SMO

The Non-RT RIC (Non-Real-Time RAN Intelligent Controller) is embedded within the SMO as a logical function. It operates on timescales greater than one second — typically minutes, hours, or longer. This is where strategic, long-horizon intelligence lives, and in an agentic architecture, this is where the highest-level planning agents reside.

The Non-RT RIC has several internal functional components that are important to understand. The Data Collection and Control (DCCS) function handles the ingestion of O1 performance data — PM files, KPI streams, configuration data — and makes it available to rApps and AI/ML workflows through a data lake or streaming interface. This is the data foundation for all analytics that happen at the Non-RT level. The ML Model Training and Inference function handles the training of AI/ML models using historical data from the data lake, manages model versioning and the model catalog, and handles the deployment of trained models either to rApps running locally or to the Near-RT RIC for xApp use. The Non-RT RIC Framework provides the service-based interface through which rApps register, discover other services, and consume data and control capabilities.

The A1 interface is how the Non-RT RIC pushes its intelligence downward to the Near-RT RIC. A1 carries three types of information. A1 Policy carries optimization policies — structured instructions to the Near-RT RIC about how to behave under specified conditions. A1 Enrichment Information (A1-EI) carries contextual data from the Non-RT RIC that helps the Near-RT RIC make better decisions — for example, a traffic prediction model output, a user mobility prediction, or a list of cells expected to be congested in the next 15 minutes. A1 Machine Learning Model Management carries trained ML models that the Non-RT RIC wants to deploy into the Near-RT RIC for xApp use, along with model performance feedback flowing back upward.

### rApps — Third-Party Intelligence in the Non-RT RIC

rApps are third-party applications that run inside the Non-RT RIC through the R1 interface. The R1 interface is the service-based interface through which rApps register themselves, discover available services from the Non-RT RIC framework, consume data services (accessing the data lake, subscribing to PM data streams), and invoke control services (triggering A1 policy generation, requesting O1 configuration changes). The rApp lifecycle has specific phases: an rApp must first onboard to the Non-RT RIC through a registration process where it declares its capabilities and data requirements, then it activates and begins consuming services, and it can be deactivated or terminated by the Non-RT RIC framework.

The RAN Intelligent Controller paper (ScienceDirect 2024) implemented and evaluated rApps for energy efficiency, interference management, and predictive maintenance on a real private 5G network testbed. The paper found that ML-based models with sequence input outperform models that use only instance input, and that the best ML models achieved more than 90% accuracy across all applications with inference time under one second.

Concrete rApp use cases illustrate how the Non-RT RIC data flows translate into network actions. An energy saving rApp accesses the data lake to retrieve weeks of historical per-cell traffic data, trains a time-series model to predict which cells will be low-traffic at what hours, and generates A1 policies defining the conditions under which specific cells may be switched off. A predictive maintenance rApp monitors long-term trends in KPIs from each cell via O1 PM data — watching for subtle degradation signatures in CRC error rates, SINR drift, or hardware alarm frequencies — and raises predicted failure alerts before actual outages occur. A traffic steering rApp identifies persistent load imbalances across cells and generates A1 policies that guide Near-RT RIC xApps to redistribute UE connections toward underloaded cells.

### Non-RT RIC as the Home of the Agentic Orchestrator

In an agentic O-RAN architecture, the Non-RT RIC is where the highest-level agentic orchestrator runs. The Agentic AI-RAN paper (arXiv:2602.24115) describes the Non-RT RIC level as operating the slowest timing-aware loop — the planning agent here works on minutes-to-hours cycles. It translates business-level intent from the operator into technical goals, manages the lifecycle of lower-level agents running in the Near-RT RIC, and maintains the long-term episodic memory that accumulates experience across many decision cycles. The Non-RT RIC's data lake becomes the agent's memory store. The A1 interface becomes the channel through which the planning agent's decisions propagate to the execution agents below. The O1 interface becomes both an observation channel (the agent reads configuration state and PM data) and an action channel (the agent can push configuration changes to network functions).

### Near-RT RIC and xApps — The Fast Control Loop

The Near-RT RIC operates on 10 millisecond to 1 second control loops. It connects to the O-CU and O-DU via the E2 interface, which operates in two directions.

In the subscription direction, an xApp tells the Near-RT RIC what measurements it wants from which cells at what frequency. The RIC passes this subscription to the O-CU/O-DU via E2, and the gNB starts sending the requested measurements. Measurement types are defined by E2 Service Models. E2SM-KPM (Key Performance Metrics) lets an xApp subscribe to UE-level and cell-level performance metrics — DL/UL throughput, SINR, RSRP, CQI, BLER, PRB utilization, handover statistics — in near real time. E2SM-RC (RAN Control) is used for the control direction.

In the control direction, the xApp sends control messages via E2SM-RC to directly change RAN parameters on the gNB. What xApps can control includes: handover A3 offset and time-to-trigger, per-UE scheduling priority weights, beam selection for specific UEs, PRB allocation between slices, DRX parameter configuration, and carrier aggregation activation or deactivation.

A key challenge in deployments with multiple xApps is conflict detection. If xApp A wants to increase the A3 offset for a cell and xApp B simultaneously wants to decrease it, the Near-RT RIC needs a conflict mediation mechanism to resolve this before sending contradictory commands to the gNB. In an agentic architecture, this conflict mediation is handled by the coordination logic between execution agents — the self-management gates defined in the Agentic AI-RAN paper are one mechanism for this.

### The Full Control Loop — How All Layers Work Together

The complete chain from the SMO down to the radio hardware can be understood through a concrete energy saving flow. The Non-RT RIC rApp (or planning agent) accesses the data lake through DCCS, analyzes historical per-cell traffic data across time of day and day of week, identifies cells that consistently carry very low traffic during specific hours, and generates an A1 Policy specifying the conditions under which those cells may go to sleep. That policy is pushed via the A1 interface to the Near-RT RIC. The energy saving xApp (or execution agent) in the Near-RT RIC receives the A1 policy and registers it as an operating constraint. It subscribes to PRB utilization of those cells via E2SM-KPM. When utilization drops below the threshold for the required consecutive duration, the xApp sends an E2SM-RC control message to the O-DU to deactivate the carrier. Before the predicted busy period, the xApp reactivates the carrier. KPI outcomes are collected via O1 PM files and fed back into the Non-RT RIC data lake, where the planning agent uses them in its next Reflect phase to evaluate whether the policy worked and whether it should be adjusted.

This is the closed loop — from long-horizon analytics and planning in the Non-RT RIC, through A1 policy, to near-real-time enforcement by the Near-RT RIC xApp, through E2 control to the actual RAN hardware, and back through O1 PM data to the Non-RT RIC. Every layer of this loop has a corresponding agentic AI component in an agentic O-RAN architecture.

---

## Part 3 — AI-RAN

### What AI-RAN Means

AI-RAN is a different concept from O-RAN's AI/ML. O-RAN AI/ML uses intelligence to control an existing 5G network. AI-RAN means running 5G baseband processing and AI inference workloads on the same physical compute hardware simultaneously.

Traditional 5G BBUs ran on dedicated ASICs or FPGAs — purpose-built chips optimized for specific signal processing tasks like LDPC encoding/decoding, FFT/IFFT, MIMO precoding. These chips are extremely fast and power-efficient for those specific tasks, but they cannot do anything else. You cannot run a language model on an LDPC accelerator.

The AI-RAN Alliance (2024 Vision Whitepaper, available at ai-ran.org) defines AI-RAN as replacing or supplementing those dedicated chips with general-purpose GPU compute. Modern GPUs are fast enough to run the full 5G L1 stack in software — NVIDIA's cuBB (CUDA Baseband Library) does exactly this — while simultaneously having idle compute capacity that can be used for AI inference workloads. The value proposition is that a cell site's compute sits mostly idle during off-peak hours. With AI-RAN, during those idle hours the same GPU hardware can serve AI inference requests — generating revenue from compute that was previously doing nothing. During peak traffic hours, the scheduler dynamically shifts more compute to the 5G baseband and less to AI workloads.

### NVIDIA's Platform

The Interplay of AI-and-RAN paper (arXiv:2503.07420) specifically analyzes GPU compute sharing between 5G and AI workloads for a converged 6G platform. NVIDIA ARC (AI-RAN Controller) is the intelligence layer that decides how to share GPU compute between the 5G baseband and AI inference workloads. It must guarantee that L1 processing latency requirements are always met — L1 must complete within 3GPP timing requirements, typically a few hundred microseconds from a subframe boundary — while using remaining capacity for AI inference without interference. NVIDIA's ARC-Compact (announced May 2025) is designed specifically for cell site deployment in D-RAN scenarios where a compact unit at the cell site runs both 5G and AI simultaneously. Spectrum-X is NVIDIA's networking platform for connecting GPU-equipped cell sites or centralized AI-RAN clusters with high bandwidth and low latency. BlueField DPUs handle fronthaul packet processing without consuming CPU or GPU cycles, freeing the main GPU for baseband and AI workloads.

### C-RAN vs D-RAN for AI-RAN

In D-RAN, every cell site has its own compute. The advantage is no fronthaul latency concern since compute is co-located with the radio. The disadvantage is that each site's GPU is underutilized most of the time, resources cannot be pooled across sites, and managing many separate GPU servers is operationally expensive.

In C-RAN, baseband processing is pooled in a central hub that serves many O-RUs. Resource utilization is much better because traffic from different cells peaks at different times — pooling smooths demand. The disadvantage is the requirement for ultra-low latency fronthaul fiber between the O-RUs and the central hub.

The direct connection between AI-RAN hardware and agentic AI software is this: an agentic system in the O-RAN SMO or Non-RT RIC needs compute to run LLM inference. If that inference runs in a remote cloud, round-trip latency adds hundreds of milliseconds to every reasoning step, costs are high, and sensitive network KPI data leaves the operator's infrastructure. With AI-RAN, the GPU compute needed for the agentic LLM is co-located with the RAN within the operator's own infrastructure. This is what makes real-time on-premises agentic RAN control technically and economically feasible.

---

## Part 4 — Agentic AI Applied to RAN

### Where Existing O-RAN AI Falls Short

Most existing O-RAN AI/ML work uses a narrow approach: train an RL model offline, deploy it as an xApp, subscribe to E2SM-KPM metrics, use those metrics as the RL state to select control actions. This works within its limits, but the limits are significant.

The Agentic AI-RAN paper (arXiv:2602.24115) states the core problem directly: O-RAN exposes rich control and telemetry interfaces across the Non-RT RIC, Near-RT RIC, and distributed units, but also makes it harder to operate multi-tenant, multi-objective RANs in a safe and auditable manner. Multi-tenant and multi-objective — this is exactly where RL xApps fail. An RL xApp optimizes one specific KPI. When there are multiple tenants, multiple objectives, and those objectives conflict with each other, RL alone is not sufficient.

The ReAct paper (arXiv:2210.03629) demonstrated that the LLM-based agentic approach outperformed imitation and reinforcement learning methods by an absolute success rate of 34% on the ALFWorld benchmark, using only one or two in-context examples. This shows that the agentic approach works in zero-shot or few-shot settings where RL would need extensive training data and environment interaction.

### The Agentic AI-RAN Architecture

The Agentic AI-RAN paper (arXiv:2602.24115) defines four agentic primitives specifically for O-RAN.

The first is the Plan-Act-Observe-Reflect loop. The paper describes it as a timing-aware control cycle aligned with the Non-RT, Near-RT, and RT layers. Timing-aware is critical — the agent at the Non-RT RIC level operates with a minutes-scale loop while the agent at the Near-RT RIC level must stay within the 10ms to 1s constraint. A single generic loop cannot serve both.

The second primitive is Skills as Tool-Use. In O-RAN, a skill is a thin, verifiable wrapper around a controllable primitive that the agent invokes as a tool. Verifiable means the skill's behavior and effect can be audited automatically — which is necessary when autonomous agents are making changes to live network configurations.

The third primitive is Memory and Evidence. Memory spans short-term caches at the Near-RT level, episodic logs per decision, and long-term knowledge curated at the SMO/Non-RT level. Evidence is generated by design at each commit — decision-level records link goals, compact context, selected actions, and observed outcomes, supporting audit and regulatory reporting without exposing raw subscriber data.

The fourth primitive is Self-Management Gates. This mechanism controls when the agent acts autonomously and when it escalates to a human. Actions with impact scores above a defined threshold require human confirmation before execution. The paper showed in a multi-cell O-RAN simulation that an agent with these four primitives improved slice life-cycle and RRM performance compared to conventional ML/RL xApps.

### RAN Cortex — Solving the Memory Problem

The RAN Cortex paper (arXiv:2505.07842) is specifically about giving O-RAN agents persistent memory. Its abstract states the problem: existing AI-based RAN decision systems remain fundamentally stateless, treating each decision as isolated. The paper proposes a memory-augmented architecture with four components: a context encoder that transforms network state into high-dimensional embeddings, a vector-based memory store of past network episodes, a recall engine that retrieves semantically similar past situations using cosine similarity or learned kernel metrics over the embedding space, and a policy interface that supplies historical context to agents in real time. The paper identifies three concrete benefits: improved sample efficiency from learning past cases without retraining, contextual generalization through retrieved memory enabling adaptation to similar unseen states, and temporal awareness for recognizing recurring patterns like peak traffic times or stadium events. The paper demonstrates these through two use cases: stadium traffic mitigation and mobility management in drone corridors.

### Edge Agentic AI and MobiLLM

The Edge Agentic AI Framework paper (arXiv:2507.21696) implements AutoGen-based multi-agent coordination at the O-RAN edge. AutoGen (arXiv:2308.08155) is Microsoft's framework where multiple LLM agents have structured conversations to solve problems. This paper shows that generalist AI frameworks with telecom-specific customization are effective for O-RAN use cases, and it references real-world deployments by NVIDIA and SoftBank to anchor the academic work in what is actually happening in industry.

The MobiLLM paper (arXiv:2509.21634) addresses the security angle. When an agentic AI autonomously changes network configurations, it also creates an attack surface. If E2 KPI data is compromised — or a fake gNB inserts false measurements — the agent's decisions can be manipulated. MobiLLM proposes an agentic framework that handles closed-loop threat detection and mitigation in 6G Open RANs, specifically addressing how an LLM-based agent detects threats, plans responses, and mitigates them automatically while keeping the process safe and auditable.

---

## Part 5 — 6G and AI-Native Networks

### AI-Assisted vs AI-Native — The Critical Distinction

Understanding 6G requires understanding this distinction clearly, because it defines the difference between what 5G O-RAN does today and what 6G will do fundamentally differently.

In the 5G O-RAN approach, the network protocol stack is designed exactly as 3GPP specified — the NR air interface, the protocol layers (PHY, MAC, RLC, PDCP, RRC), the handover procedures, the scheduling algorithms all work as standardized. AI and ML are deployed on top as an optimization layer — reading measurements, adjusting parameters, making configuration changes through defined interfaces. AI is an add-on. Remove it and the network still functions, just less optimally. This is AI-assisted design.

In AI-native 6G design, the protocol stack itself has AI as an integral component. AI does not adjust parameters of a human-designed protocol — AI is part of the protocol itself. Several specific examples show what this means at the air interface level.

AI-native channel estimation replaces the pilot-based approach used in 5G NR. In 5G, DMRS (Demodulation Reference Signals) are transmitted at known time-frequency positions. The receiver estimates the channel at those pilot positions by comparing received signals to known transmitted signals, then interpolates across the full bandwidth and time duration. This entire procedure is a fixed signal processing algorithm designed by human engineers. In 6G AI-native design, a neural network performs channel estimation end-to-end — trained on actual channel data, it learns to extract channel state in ways that outperform human-designed algorithms particularly in environments that traditional estimators model poorly, such as high-mobility scenarios, heavily scattered urban channels, or very large antenna arrays.

AI-native CSI feedback transforms how UEs report channel state to the gNB for massive MIMO beamforming. In 5G, the UE measures the channel and compresses its report using codebooks — structured quantization schemes that 3GPP defined. In 6G, the compression is performed by a trained autoencoder: the encoder runs at the UE and compresses the channel measurement into a compact latent representation, and the decoder runs at the gNB and reconstructs the channel from the compressed feedback. This encoder-decoder pair is trained end-to-end jointly, meaning it learns the most information-preserving compression for the actual channel statistics the network encounters, rather than a fixed codebook designed for an idealized channel model. 3GPP Release 18 is already studying this in its AI/ML for NR air interface work item.

AI-native beam management addresses a major overhead problem in mmWave (millimeter wave) deployments. Beam sweeping in 5G NR exhaustively transmits reference signals across many beam directions and waits for UEs to report the best beam pair — this takes time and consumes significant radio resources. An AI-native approach predicts the best beam based on UE position history, past beam quality measurements, surrounding cell configurations, and environmental context learned during training. The beam is selected predictively rather than reactively, dramatically reducing sweeping overhead.

AI-native scheduling embeds AI directly into the L2 MAC scheduler. Instead of a human-designed scheduling algorithm that uses simple heuristics (proportional fair, max-min fair, round robin), an AI scheduler learns to allocate PRBs across users in ways that optimize a learned representation of the network objective. The scheduler is not adjusted by AI — it is an AI.

### 3GPP Roadmap Toward AI-Native

3GPP is approaching AI-native design incrementally rather than in a single jump, because every change to the standard must be backward compatible with deployed equipment and tested across the entire ecosystem.

Release 18 (5G Advanced) is the first release where AI/ML appears in the NR specification itself. Three work items are active: AI/ML for air interface, which covers CSI feedback compression, beam management enhancement, and positioning accuracy improvement as described above; AI/ML for RAN intelligence, which covers the framework for how ML models are managed, trained, and deployed within the RAN; and AI/ML for network data analytics, extending the NWDAF (Network Data Analytics Function) in the 5G core.

Release 19 is expanding AI/ML coverage. Studies are underway on AI/ML for the air interface in additional scenarios beyond those in Release 18, AI/ML for the 5G core network functions, and — significantly — the study of AI/ML model lifecycle management as a first-class network function. This last item is directly relevant to agentic AI: if the network has a standardized function for managing ML model training, versioning, and deployment, then an agentic orchestrator has a standardized interface through which to manage its own models and the models of other agents.

Release 20 and beyond, aligned with ITU-R IMT-2030 (the international 6G framework), targets the full AI-native architecture. The IMT-2030 framework explicitly lists AI and ML as fundamental technology components of 6G — not optional enhancements but integral to the system design.

### What IMT-2030 Actually Requires

The ITU-R IMT-2030 Recommendation M.2160 establishes the framework for 6G. Beyond simply naming AI as important, it defines specific capabilities that are relevant here. The framework requires that 6G networks support native AI and ML as part of the network architecture — meaning the network should be able to train models, deploy them, monitor their performance, and update them through standardized network functions. It requires AI-native air interfaces that can adapt to channel conditions using learned models rather than fixed algorithms. It requires networks that can self-configure and self-optimize, which is the standardization path toward what this repo calls agentic AI.

### Bharat 6G Vision

India's Department of Telecommunications published the Bharat 6G Vision Document in 2023. The document calls for India to be a 6G technology creator rather than just a consumer, with a goal of owning at least 10% of global 6G patents — a significant shift from India's historically adoption-focused role in telecom standardization. The vision explicitly identifies AI-native design as a priority: the network should be designed with AI as a core component from the ground up, not retrofitted after the fact. Energy efficiency — specifically a target of significantly better energy per bit than 5G — and self-configuring, self-optimizing, self-healing network capabilities are also explicitly stated requirements. TSDSI (Telecom Standards Development Society India) is the standards body coordinating India's contributions to 3GPP and ITU-R for 6G.

---

## Part 6 — Agentic Design Patterns

### Why Patterns Matter

The Agentic AI-RAN paper (arXiv:2602.24115) formalizes O-RAN-specific agentic primitives that are essentially telecom-specific design patterns. The Agentic Design Patterns paper (arXiv:2601.19752) by Belcak et al. provides a formal taxonomy of general patterns. Reading both together shows how each general pattern maps onto specific O-RAN layers, interfaces, and timing constraints. Understanding the patterns means that when you read a new paper, you can immediately identify which patterns it uses and which it is missing.

### Chain of Thought

Chain of Thought is the foundation of all agentic reasoning. The core insight was that asking an LLM to reason step by step before answering dramatically improves accuracy on complex problems. Each reasoning step conditions the next, building toward a more reliable conclusion rather than making a leap that skips necessary logic. In the context of O-RAN, CoT operates at every layer where an agent reasons before acting — it is the mechanism behind the Thought step in every ReAct loop instance, whether that loop is running at the Non-RT RIC level on a minutes-scale cycle or at the Near-RT RIC level on a seconds-scale cycle.

### ReAct

The ReAct paper (arXiv:2210.03629) describes the pattern as generating both verbal reasoning traces and actions in an interleaved manner, allowing the model to perform dynamic reasoning to create, maintain, and adjust high-level plans for acting, while also interacting with external environments to incorporate additional information into reasoning. In O-RAN, the external environments are the O-RAN interfaces themselves — E2SM-KPM for observations, E2SM-RC for actions at the Near-RT level, A1 for policy propagation, O1 for management observations and actions. The ReAct pattern is how the agent turns these interface interactions into a coherent reasoning loop rather than isolated API calls.

### Reflection

Reflection adds a structured self-critique step after each action-observation cycle. Rather than immediately generating the next action after observing an outcome, the agent first evaluates: was this the right action? Did it achieve what was intended? What evidence contradicts my expectations? The Agentic AI-RAN paper implements reflection as the Observe-Reflect phase at each timing layer. At the Non-RT RIC level, reflection happens on A1 policy outcomes — the agent checks whether the KPI trends that were predicted before pushing the policy actually materialized after it was enforced. At the Near-RT RIC level, reflection happens on E2SM-RC control outcomes — the agent checks whether the gNB parameter change produced the expected KPI change within the expected timeframe. Reflection is not just internal reasoning — the Agentic AI-RAN paper's evidence logging means every reflection cycle is a documented record that can be externally audited.

### Planning and Decomposition

For complex, multi-step goals, a pure ReAct loop can lose track of the high-level objective while handling individual tool calls. The planning pattern adds an explicit upfront step where the agent generates a complete plan — a numbered sequence of sub-goals and their expected outcomes — before beginning execution. During execution, the plan is tracked and updated when new observations require it. The Agentic AI-RAN paper aligns planning explicitly with O-RAN timing constraints: at the Non-RT RIC level, a plan covers minutes-to-hours and involves selecting which cells to target, which A1 policies to generate, and which rApps to activate. At the Near-RT RIC level, a plan covers seconds and involves which E2SM-KPM subscriptions to set up, which threshold conditions to watch for, and which E2SM-RC control messages to send in which sequence.

### Multi-Agent Orchestration

The Agentic AI for Intent-driven Optimization paper (arXiv:2602.22539) implements multi-agent orchestration concretely in cell-free O-RAN, as described in Part 1. The orchestration pattern in O-RAN maps naturally onto the existing hierarchy: the supervisor agent at the SMO/Non-RT RIC level sets goals and constraints, specialist agents at the Non-RT RIC level handle specific optimization domains, and execution agents at the Near-RT RIC level carry out the fast control actions. The key insight from the paper is parameter-efficient fine-tuning that allows the same underlying LLM to power multiple specialized agents — each agent sees a different system prompt and memory context, but shares the same model weights. This makes the multi-agent system economically viable for deployment at scale.

### Memory-Augmented Retrieval

The RAN Cortex paper (arXiv:2505.07842) formalizes this pattern for O-RAN. The paper notes that user behavior, traffic load, interference, and mobility exhibit episodic structure — recurrent patterns that reappear across space and time. Memory-augmented retrieval is the mechanism for exploiting this structure. The recall engine retrieves semantically similar past network situations from the episodic memory store using cosine similarity or learned kernel metrics over the high-dimensional embedding space. The retrieved episodes are then supplied to the policy interface, which conditions the agent's current decision on both the live current state and the recalled prior experience. This is why the agent improves over time without retraining — it is learning by analogy from its own history rather than gradient updates.

### Constitutional Constraints and Self-Management Gates

The Agentic AI-RAN paper's self-management gates implement the constitutional constraints pattern specifically for O-RAN. The key design decision is that constraints are hard-coded independently of the agent's reasoning — they define boundaries of the action space that the agent can never cross regardless of what its planning produces. This independence from the AI is deliberate: you cannot have a safety boundary that the AI itself can reason its way around. In O-RAN terms, these constraints might specify that the agent cannot simultaneously deactivate more than a defined number of cells, cannot modify transmit power below a coverage threshold, cannot change handover parameters by more than a defined delta in a single action, or cannot make any configuration changes during defined maintenance freeze windows.

Actions that pass the constraint check but still have impact scores above a secondary threshold trigger automatic human escalation rather than autonomous execution. This creates a two-tier safety system: hard constraints that block execution completely, and soft impact thresholds that pause execution for human review. The combination allows autonomous operation on the majority of lower-impact decisions while preserving human authority over decisions that could have wide-area network effects. This maps directly onto how operational teams in telecom actually work — routine parameter adjustments are automated, but anything that could affect a significant number of subscribers requires a change management approval.

---

## Part 7 — Governance Safety and Ethics

### Why This Is Non-Negotiable in Telecom

The MobiLLM paper (arXiv:2509.21634) opens with this point explicitly: deploying agentic AI in 6G O-RAN creates security and safety challenges that were not present in traditional ML deployments. A misconfigured beam or an incorrect handover parameter change can cause service outages affecting thousands or millions of users, including emergency services and critical infrastructure.

The Agentic AI-RAN paper (arXiv:2602.24115) makes the multi-tenant dimension explicit: in a shared O-RAN infrastructure serving multiple operators' network slices, one operator's agentic controller can inadvertently affect another operator's slice. This is not a hypothetical risk — it is an architectural consequence of the shared control plane.

### The Risk Landscape

Cascading failures arise because in a densely interconnected network, an optimization in one cell affects neighboring cells. An agent that optimizes locally without modeling the impact on neighboring cells can create an interference chain that oscillates and amplifies rather than converges. This is agent coordination failure — individually rational actions by multiple agents produce a collectively irrational outcome.

Distribution shift means an agent optimized on one period of network data will fail when conditions change. The agent has no way to detect that its learned patterns no longer apply to the current environment.

Goal misalignment occurs when the agent's formal objective does not capture all the dimensions that matter. An agent instructed to maximize throughput will do exactly that — even if doing so depletes spectrum reserved for QoS-sensitive traffic, drives energy consumption up, or creates interference in neighboring spectrum bands. Telecom goals are multi-dimensional and the formal objective must reflect all of them.

Adversarial inputs are a specific risk in O-RAN because the agent's observations arrive via E2 KPI subscriptions from the gNB. If a gNB is compromised or a fake gNB is inserted into the network, it can send false KPI measurements that manipulate the agent's decisions. MobiLLM (arXiv:2509.21634) specifically addresses this threat model with a closed-loop detection and response mechanism.

### Technical Safety Mechanisms

The Agentic AI-RAN paper's self-management gates implement guardrails that constrain the action space. Actions outside defined safe ranges are rejected before execution, deterministically and independently of the agent's reasoning — so even if the planning module produces a recommendation outside the safe boundary, it will not reach the network.

Shadow mode deployment is the standard path for introducing a new agent to a live network. The agent subscribes to real data and generates recommendations, but nothing is executed. Recommendations are logged and reviewed. The operator validates behavior before enabling autonomous mode.

The RAN Cortex paper's episodic memory architecture provides natural support for automatic rollback. Every action generates an episodic log entry with the pre-action network state, the action taken, the predicted outcome, and the actual outcome. When KPI monitoring detects degradation after an action, the pre-action configuration can be restored automatically from the log.

### Regulatory and Policy Context

NIST's AI Risk Management Framework (2023) and the EU AI Act — which classifies systems making consequential decisions in critical infrastructure as high-risk — both explicitly require explainability, auditability, and continuous monitoring for such systems. Telecom qualifies under these definitions. The ReAct loop's reasoning trace provides the natural basis for explainability — every decision has a logged chain of reasoning. The Agentic AI-RAN paper's evidence generation mechanism provides the auditability layer. Continuous monitoring frameworks and incident reporting specific to autonomous RAN AI remain an active research gap that no paper in this reading list has fully solved — this is an open research problem.

---

## Papers Reading List

All arXiv links are permanent and free to access.

### Foundational Surveys

| Paper | Link |
|-------|------|
| Understanding O-RAN: Architecture, Interfaces, Algorithms, Security — Polese et al. 2023 | https://arxiv.org/abs/2202.01032 |
| ReAct: Synergizing Reasoning and Acting in Language Models — Yao et al. 2022 | https://arxiv.org/abs/2210.03629 |
| Cognitive Architectures for Language Agents (CoALA) — Wang et al. 2023 | https://arxiv.org/abs/2309.02427 |
| Towards Trustworthy Agentic AI: A Comprehensive Survey 2025 | https://arxiv.org/abs/2502.09920 |
| From System 1 to System 2: A Survey of Reasoning LLMs 2025 | https://arxiv.org/abs/2502.17419 |

### Core Agentic AI and RAN Papers

| Paper | Link |
|-------|------|
| Agentic AI-RAN: Intent-Driven, Explainable, Self-Evolving Open RAN 2026 | https://arxiv.org/abs/2602.24115 |
| RAN Cortex: Memory-Augmented Intelligence for Context-Aware O-RAN 2025 | https://arxiv.org/abs/2505.07842 |
| Edge Agentic AI Framework for Autonomous Network Optimisation in O-RAN 2025 | https://arxiv.org/abs/2507.21696 |
| MobiLLM: Agentic AI for Closed-Loop Threat Mitigation in 6G Open RANs 2025 | https://arxiv.org/abs/2509.21634 |
| Agentic AI for Intent-driven Optimization in Cell-free O-RAN 2026 | https://arxiv.org/abs/2602.22539 |
| The Interplay of AI-and-RAN: Dynamic Resource Allocation for Converged 6G 2025 | https://arxiv.org/abs/2503.07420 |
| Agentic AI meets NAS: Proactive Traffic Prediction for AI-RAN 2025 | https://arxiv.org/abs/2510.00851 |
| Wireless Large AI Model: Shaping the AI-Native Future of 6G 2025 | https://arxiv.org/abs/2504.14653 |

### Agentic AI Frameworks and Design

| Paper | Link |
|-------|------|
| Enhancing AI Systems with Agentic Workflow Patterns 2026 | https://arxiv.org/abs/2601.12560 |
| Agentic Design Patterns — Belcak et al. 2026 | https://arxiv.org/abs/2601.19752 |
| AutoGen: Enabling Next-Gen LLM Apps via Multi-Agent Conversation 2023 | https://arxiv.org/abs/2308.08155 |
| Agentic AI: A Comprehensive Survey of Architectures and Applications 2025 | https://arxiv.org/abs/2510.25445 |

---

## Reading Order

**Step 1** — Read arXiv:2202.01032 (Polese et al. O-RAN survey). This is the foundation for everything. Focus on the disaggregation of O-RU, O-DU, and O-CU; the interface table covering A1, E2, O1, O2, and R1; and the RIC architecture distinguishing Non-RT from Near-RT and rApps from xApps. Pay specific attention to the SMO section — understanding what the SMO actually does internally is critical for Part 2 of this README.

**Step 2** — Read arXiv:2210.03629 (ReAct paper). Short paper, read it fully. It is the foundation of all agentic AI. Note the specific quantitative results: 34% absolute improvement over RL on ALFWorld, and the demonstrated reduction in hallucination on HotpotQA. These numbers serve as reference points when evaluating performance claims in the RAN papers later.

**Step 3** — Read arXiv:2309.02427 (CoALA — Cognitive Architectures for Language Agents). Focus on Sections 2 and 3 covering the memory taxonomy and agent architecture classification. This is the formal source for the four memory types discussed in Part 1.

**Step 4** — Read arXiv:2602.24115 (Agentic AI-RAN paper). This is the central paper for everything this repo is about. Note three things specifically: the four agentic primitives and exactly how they map to O-RAN layers and their respective interfaces; the self-management gates mechanism and how the impact scoring works; and the multi-cell simulation results comparing against conventional ML/RL xApps.

**Step 5** — Read arXiv:2505.07842 (RAN Cortex). Focus on the memory architecture for O-RAN: what problem is identified (stateless agents), the four-element architecture proposed as a solution, how the recall engine works over the embedding space, and how improvement is demonstrated in the stadium traffic mitigation and drone corridor use cases.

**Step 6** — Read arXiv:2507.21696 and arXiv:2509.21634 (Edge Agentic AI and MobiLLM). These are complementary to Step 4 — the edge deployment angle and the security/threat mitigation angle respectively. Note specifically how MobiLLM's threat model addresses the adversarial input risk identified in Part 7.

**Step 7** — Read arXiv:2503.07420 (Interplay of AI and RAN). This is the AI-RAN hardware side — GPU compute sharing between 5G and AI workloads. After this paper, the C-RAN vs D-RAN tradeoffs in Part 3 and the connection between AI-RAN hardware and agentic AI software feasibility will become much more concrete.

**Step 8** — Read arXiv:2602.22539 (Agentic AI for Intent-driven Optimization in Cell-free O-RAN). Concrete multi-agent implementation with specific quantitative results. Note the 4x O-RU count reduction result and the parameter-efficient fine-tuning approach that makes running multiple agents economically feasible.

**Step 9** — Read arXiv:2601.19752 and arXiv:2601.12560 (Agentic Design Patterns and Workflow Patterns). Having read all the application papers, the formal taxonomy in these papers will be much more meaningful — you will recognize which patterns each RAN paper was using and which gaps each paper left.

**Step 10** — Read arXiv:2502.09920 (Trustworthy Agentic AI) and arXiv:2502.17419 (Reasoning LLMs survey). The governance and safety foundation, and the mechanistic explanation for why modern LLMs can do the step-by-step reasoning that agentic systems require.

---

## Glossary

| Term | Definition |
|------|-----------|
| **A1 interface** | Interface from the Non-RT RIC down to the Near-RT RIC. Carries three types of content: A1 Policy (optimization constraints and guidance), A1 Enrichment Information (contextual data like traffic predictions), and ML Model Management (model deployment and feedback). Operates on seconds to minutes timescales. |
| **A1-EI** | A1 Enrichment Information. Contextual data flowing from the Non-RT RIC to the Near-RT RIC via A1 to help xApps make better decisions — for example, a traffic prediction for the next 15 minutes or a list of cells expected to become congested. |
| **AI-native** | Network design where AI is integral to how protocols work, not an optional add-on. In 6G this means AI performing channel estimation, CSI compression via autoencoders, beam management prediction, and scheduling at the protocol level. |
| **AI-RAN** | Running 5G baseband processing and AI inference workloads on the same GPU hardware at the cell site or central hub simultaneously. The hardware convergence story, separate from the software intelligence of O-RAN. |
| **Agentic AI** | An AI system that has a goal, makes a plan to reach it, uses tools to take real actions, observes the results, reflects on outcomes, and continues until the goal is reached without a human directing each step. |
| **AutoGen** | Microsoft's open-source framework for multi-agent systems where multiple LLM agents have structured conversations to solve problems collaboratively. Used in several O-RAN agentic papers as the coordination layer. |
| **C-RAN** | Centralized RAN. Baseband processing is pooled in a central hub serving many remote radio heads. Enables large GPU cluster pooling for AI-RAN but requires ultra-low latency fronthaul fiber between O-RUs and the hub. |
| **Chain of Thought** | Prompting an LLM to reason step by step before producing an answer. Forces intermediate reasoning steps that improve accuracy on complex problems. The basis of the Thought step in every agentic reasoning loop. |
| **CoALA** | Cognitive Architectures for Language Agents — Princeton 2023 paper (arXiv:2309.02427) that formally categorizes AI agent memory into four types drawn from cognitive science: in-context, episodic, semantic, and procedural. |
| **Constitutional constraints** | Hard rules the agent can never violate regardless of its reasoning output. Checked deterministically before any action executes. Implemented independently of the AI to guarantee they cannot be reasoned around. |
| **cuBB** | NVIDIA CUDA Baseband Library. Software implementation of the full 5G L1 stack running on GPU, enabling the same hardware to run both 5G baseband processing and AI inference workloads simultaneously. |
| **D-RAN** | Distributed RAN. Compute at each individual cell site. Simpler fronthaul requirements but no resource pooling across sites, with each site's GPU underutilized much of the time. |
| **DCCS** | Data Collection and Control Service. The functional component within the SMO/Non-RT RIC that ingests O1 performance data — PM files, KPI streams, configuration state — and makes it available to rApps and AI/ML workflows through a data lake or streaming interface. |
| **E1 interface** | Between O-CU-CP (Control Plane) and O-CU-UP (User Plane) within the O-CU function. |
| **E2 interface** | Between the Near-RT RIC and O-CU/O-DU. The interface through which xApps subscribe to KPI measurements via E2SM-KPM and send control commands to the gNB via E2SM-RC. The most important interface for understanding what xApps can actually do. |
| **E2SM-KPM** | E2 Service Model for Key Performance Metrics. Used by xApps to subscribe to near-real-time UE-level and cell-level performance measurements from the gNB including throughput, SINR, RSRP, CQI, BLER, PRB utilization, and handover statistics. |
| **E2SM-RC** | E2 Service Model for RAN Control. Used by xApps to send control commands that change RAN parameters — handover thresholds, scheduling weights, beam configuration, carrier aggregation, DRX settings, and others. |
| **eCPRI** | Enhanced Common Public Radio Interface. Protocol used on the fronthaul link between O-DU and O-RU. Has strict one-way latency requirements — typically under 100 microseconds in C-RAN deployments. |
| **Episodic memory** | Records of specific past events in an agentic system — what the agent observed, what it decided, what action it took, and what outcome resulted. Retrieved using similarity search and used to guide decisions in analogous future situations. |
| **F1 interface** | Between O-DU and O-CU. F1-C for control plane signaling, F1-U for user plane data. |
| **IMT-2030** | ITU-R Recommendation M.2160, the international 6G standardization framework. Defines AI and ML as fundamental technology components of 6G — not optional enhancements but integral to the architecture. |
| **LangGraph** | LangChain framework for building multi-agent workflows as directed graphs where each node is an agent or a tool. Used for implementing multi-agent orchestration patterns with controlled state passing between agents. |
| **LLM** | Large Language Model. The reasoning backbone of modern agentic systems. Given a context of observations, retrieved memory, and goal specification, it generates the next Thought and Action in the ReAct loop. |
| **Near-RT RIC** | Near-Real-Time RAN Intelligent Controller. Operates on 10ms to 1 second control loops. Hosts xApps that directly observe the RAN via E2SM-KPM and control it via E2SM-RC. In an agentic architecture, execution agents run here on the fast timing layer. |
| **Non-RT RIC** | Non-Real-Time RAN Intelligent Controller. Embedded within the SMO. Operates on timescales greater than 1 second. Contains the DCCS data lake, ML model training and catalog functions, and the Non-RT RIC Framework that hosts rApps via R1. In an agentic architecture, planning and analytics agents run here on the slow timing layer. |
| **O-Cloud** | The cloud infrastructure — servers, storage, networking — on which O-RAN network functions run as containerized workloads, typically Kubernetes-managed. The SMO manages O-Cloud lifecycle through the O2 interface. |
| **O-CU** | Open Centralized Unit. Handles upper protocol layers: RRC, SDAP, PDCP. Split into O-CU-CP and O-CU-UP connected via E1. Connects to 5G Core via N2 (control) and N3 (user plane). |
| **O-DU** | Open Distributed Unit. Handles lower L2 — MAC and RLC — and remaining L1 including LDPC channel coding. Connects to O-CU via F1 and to O-RU via fronthaul eCPRI. |
| **O-RAN** | Open Radio Access Network. Disaggregates the traditional monolithic base station into O-RU, O-DU, and O-CU with open interfaces between them, adds a programmable intelligence layer through the SMO, Non-RT RIC, and Near-RT RIC, and enables multi-vendor interoperability and third-party AI/ML application deployment. |
| **O-RU** | Open Radio Unit. The physical radio hardware — antenna, RF electronics, and the very bottom of L1. Connected to the O-DU via the fronthaul interface using eCPRI. |
| **O1 interface** | Between SMO and O-DU/O-CU/O-RU. Used for management: configuration via NETCONF/YANG, PM data collection at defined intervals, fault alarm reception, and software lifecycle management. The configure-and-observe interface. |
| **O2 interface** | Between SMO and O-Cloud. Used for infrastructure lifecycle management: deploying, scaling, and managing containerized network function workloads on the O-Cloud Kubernetes infrastructure. |
| **Plan-Act-Observe-Reflect** | Core agentic primitive for O-RAN defined in arXiv:2602.24115. Timing-aware — the loop cadence at the Non-RT RIC level is minutes-scale, at the Near-RT RIC level is 10ms to 1s scale. A single generic loop cannot serve both. |
| **R1 interface** | Between rApps and the Non-RT RIC Framework. The service-based interface through which rApps register capabilities, discover available services, consume data from the DCCS data lake, and invoke control services including A1 policy generation. |
| **RAN Cortex** | Memory-augmented architecture for O-RAN agents proposed in arXiv:2505.07842. Four components: context encoder, vector-based memory store, recall engine using cosine similarity over embeddings, and policy interface. Adds persistent episodic memory to agents that are otherwise stateless. |
| **rApp** | Non-RT RIC Application. Third-party software running inside the Non-RT RIC via the R1 interface. Registers through an onboarding lifecycle, then accesses DCCS data and invokes Non-RT RIC Framework services. Use cases include energy saving, predictive maintenance, interference management, and traffic steering. In an agentic architecture, rApps can be agents themselves or host agents. |
| **RAG** | Retrieval-Augmented Generation. Retrieving relevant documents from an external knowledge base before generating a response. Extended in agentic systems to episodic memory retrieval — the agent retrieves similar past network events from its memory store before deciding what to do next. |
| **ReAct** | Reason plus Act. The fundamental execution loop of LLM-based agentic systems: observe the current state, generate a Thought explaining the reasoning, generate an Action calling a tool, observe the tool result, generate the next Thought, and continue until a Final Answer is reached. |
| **Reflection** | Agentic design pattern where the agent evaluates its past actions after observing outcomes — was the action correct, did it achieve the intended effect, what does the evidence say. In O-RAN implemented as the Reflect phase in the Plan-Act-Observe-Reflect loop, with automatic evidence logging making reflection an auditable record. |
| **Self-management gates** | Two-tier safety mechanism in arXiv:2602.24115. First tier: hard constitutional constraints that block any action outside the defined safe range, checked deterministically and independently of AI reasoning. Second tier: impact score threshold above which actions pause for human confirmation rather than executing autonomously. |
| **Shadow mode** | Deploying an agent such that it subscribes to real data and generates recommendations but executes nothing. Used to validate that agent behavior is correct on live network data before enabling autonomous operation. |
| **Skills** | O-RAN-specific tool concept from arXiv:2602.24115. Thin, verifiable wrappers around controllable primitives — for example, a PRB reallocation skill wraps the E2SM-RC control message needed to change PRB allocation between slices. The planning agent calls the skill by name with semantic parameters; the skill handles the protocol details. |
| **SMO** | Service Management and Orchestration. The top layer of the O-RAN management hierarchy. Contains the Non-RT RIC (including DCCS, ML model management, and the Non-RT RIC Framework), manages O-RAN network functions via O1, manages the O-Cloud via O2, and in an agentic architecture hosts the highest-level orchestrator agent. |
| **xApp** | Near-RT RIC Application. Third-party software deployed in the Near-RT RIC that subscribes to E2SM-KPM measurements from the gNB and sends E2SM-RC control commands to change RAN parameters. Operates on 10ms to 1 second timescales. In an agentic architecture, execution agents run as or within xApps. |
| **Y1 interface** | From the Near-RT RIC to external consumers. Exposes RAN analytics information to applications that want to consume network intelligence without directly controlling the RAN — for example, a capacity planning tool that wants near-real-time cell load data. |
