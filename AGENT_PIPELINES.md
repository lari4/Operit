# Agent Pipelines & Workflows

This document describes all possible agent workflows and pipelines in the Operit application, showing how prompts are used, what data flows between components, and how different systems interact.

---

## Table of Contents

1. [Standard Chat Pipeline](#1-standard-chat-pipeline)
2. [Plan Mode (Deep Search) Pipeline](#2-plan-mode-deep-search-pipeline)
3. [Memory & Knowledge Graph Pipeline](#3-memory--knowledge-graph-pipeline)
4. [Conversation Summarization Pipeline](#4-conversation-summarization-pipeline)
5. [UI Automation Pipeline](#5-ui-automation-pipeline)
6. [Voice Mode Pipeline](#6-voice-mode-pipeline)
7. [Translation Pipeline](#7-translation-pipeline)
8. [Package Description Generation Pipeline](#8-package-description-generation-pipeline)

---

## 1. Standard Chat Pipeline

**Purpose:** The primary conversational flow for general user interactions with tool support.

**Entry Point:** User sends a message in chat mode

**Key Components:**
- `ConversationService`
- `EnhancedAIService`
- `SystemPromptConfig`
- Tool execution system

### ASCII Flow Diagram

```
┌─────────────────┐
│  User Message   │
└────────┬────────┘
         │
         ▼
┌─────────────────────────────────────┐
│  ConversationService                │
│  - Build system prompt              │
│  - Apply customizations             │
└────────┬────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────┐
│  System Prompt Assembly             │
│  ┌───────────────────────────────┐  │
│  │ Base Components:              │  │
│  │ - Behavior Guidelines         │  │
│  │ - Tool Usage Guidelines       │  │
│  │ - Package System Guidelines   │  │
│  │ - Web Workspace Guidelines    │  │
│  │ - Thinking Guidance (if on)   │  │
│  └───────────────────────────────┘  │
│  ┌───────────────────────────────┐  │
│  │ Custom Components:            │  │
│  │ - Custom intro prompt         │  │
│  │ - User preference text        │  │
│  │ - Waifu rules (if enabled)    │  │
│  │ - Desktop pet mood rules      │  │
│  │ - Character card prompts      │  │
│  │ - Prompt tags                 │  │
│  └───────────────────────────────┘  │
│  ┌───────────────────────────────┐  │
│  │ Available Tools Description   │  │
│  │ - File system tools           │  │
│  │ - HTTP tools                  │  │
│  │ - Memory tools (if enabled)   │  │
│  │ - Active package tools        │  │
│  └───────────────────────────────┘  │
└────────┬────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────┐
│  EnhancedAIService                  │
│  - Append chat history              │
│  - Send to LLM provider             │
└────────┬────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────┐
│  LLM Response Stream                │
└────────┬────────────────────────────┘
         │
         ├─────────────────────┐
         │                     │
         ▼                     ▼
    ┌────────┐         ┌──────────────┐
    │  Text  │         │  Tool Call   │
    └────┬───┘         └──────┬───────┘
         │                    │
         │                    ▼
         │            ┌──────────────────┐
         │            │  Execute Tool    │
         │            │  - File ops      │
         │            │  - HTTP request  │
         │            │  - Memory query  │
         │            │  - Package use   │
         │            └──────┬───────────┘
         │                   │
         │                   ▼
         │            ┌──────────────────┐
         │            │  Tool Result     │
         │            └──────┬───────────┘
         │                   │
         │                   ▼
         │            ┌──────────────────┐
         │            │  Send to LLM     │
         │            │  (Next turn)     │
         │            └──────────────────┘
         │
         ▼
┌─────────────────────────────────────┐
│  Response Complete                  │
│  - <status type="complete">         │
└────────┬────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────┐
│  Post-Processing (if enabled)       │
│  ┌───────────────────────────────┐  │
│  │ 1. Conversation Summary       │  │
│  │    (See Pipeline 4)           │  │
│  │                               │  │
│  │ 2. Knowledge Graph Update     │  │
│  │    (See Pipeline 3)           │  │
│  └───────────────────────────────┘  │
└─────────────────────────────────────┘
```

### Data Flow

**Input:**
- User message text
- Chat history (previous conversation turns)
- User preferences (age, gender, personality, etc.)
- Enabled features (thinking mode, memory query, waifu mode, etc.)
- Workspace path (if web workspace is bound)
- Active packages and tools

**Processing:**
1. Load user preferences and build preference text
2. Assemble system prompt with all components
3. Apply custom prompts and personality settings
4. Add available tools documentation
5. Send to LLM with chat history
6. Stream response back to user
7. Execute tools if called
8. Trigger post-processing (summary, memory update)

**Output:**
- Streamed AI response text
- Tool execution results
- Updated conversation state
- Memory graph updates (async)
- Conversation summary (async)

---

## 2. Plan Mode (Deep Search) Pipeline

**Purpose:** Breaks down complex tasks into parallel/sequential subtasks for efficient multi-agent execution.

**Entry Point:** User enables plan mode or task complexity triggers it

**Key Components:**
- `PlanModeManager`
- `TaskExecutor`
- `EnhancedAIService`
- Subtask agents with specialized prompts

### ASCII Flow Diagram

```
┌─────────────────┐
│  User Request   │
│  (Complex Task) │
└────────┬────────┘
         │
         ▼
┌──────────────────────────────────────────┐
│  PlanModeManager                         │
│  - Initialize execution state            │
└────────┬─────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────┐
│  Step 1: Generate Execution Plan         │
│  ┌────────────────────────────────────┐  │
│  │ Prompt: PLAN_GENERATION_PROMPT     │  │
│  │ Input:                             │  │
│  │   - User message                   │  │
│  │   - Chat history                   │  │
│  │                                    │  │
│  │ AI Task: Decompose into subtasks   │  │
│  │                                    │  │
│  │ Output: JSON Plan                  │  │
│  │ {                                  │  │
│  │   "tasks": [                       │  │
│  │     {                              │  │
│  │       "id": "task_1",              │  │
│  │       "name": "...",               │  │
│  │       "instruction": "...",        │  │
│  │       "dependencies": []           │  │
│  │     },                             │  │
│  │     ...                            │  │
│  │   ],                               │  │
│  │   "final_summary_instruction": "..." │
│  │ }                                  │  │
│  └────────────────────────────────────┘  │
└────────┬─────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────┐
│  Step 2: Parse Plan & Build Graph        │
│  - Create ExecutionGraph                 │
│  - Identify task dependencies            │
│  - Determine execution order             │
└────────┬─────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────┐
│  Step 3: Execute Tasks (TaskExecutor)    │
│  - Tasks without dependencies run first  │
│  - Parallel execution when possible      │
└────────┬─────────────────────────────────┘
         │
         ├─────────────┬─────────────┬──────────────┐
         │             │             │              │
         ▼             ▼             ▼              ▼
    ┌────────┐   ┌────────┐   ┌────────┐   ┌────────┐
    │ Task 1 │   │ Task 2 │   │ Task 3 │   │ Task N │
    │(agent) │   │(agent) │   │(agent) │   │(agent) │
    └───┬────┘   └───┬────┘   └───┬────┘   └───┬────┘
        │            │            │            │
        │            │            │            │
        │    Each Task Execution Process:      │
        │    ┌─────────────────────────┐       │
        │    │ 1. Build Task Context   │       │
        │    │    - Original request   │       │
        │    │    - Task name          │       │
        │    │    - Dependency results │       │
        │    └───────────┬─────────────┘       │
        │                │                     │
        │                ▼                     │
        │    ┌─────────────────────────┐       │
        │    │ 2. Build Full Instruction│      │
        │    │    Template:            │       │
        │    │    ---                  │       │
        │    │    [context]            │       │
        │    │                         │       │
        │    │    请根据以上上下文信息,  │       │
        │    │    执行以下具体任务:      │       │
        │    │    [task.instruction]   │       │
        │    │                         │       │
        │    │    请专注于完成这个特定的 │       │
        │    │    子任务，你的回答将作为 │       │
        │    │    整个计划的一部分。     │       │
        │    └───────────┬─────────────┘       │
        │                │                     │
        │                ▼                     │
        │    ┌─────────────────────────┐       │
        │    │ 3. Create Subtask Agent │       │
        │    │    System Prompt:       │       │
        │    │    - SUBTASK_AGENT      │       │
        │    │      _PROMPT_TEMPLATE   │       │
        │    │    - Strict efficiency  │       │
        │    │      mandates           │       │
        │    │    - No wait_for_user   │       │
        │    │    - Parallel tools only│       │
        │    └───────────┬─────────────┘       │
        │                │                     │
        │                ▼                     │
        │    ┌─────────────────────────┐       │
        │    │ 4. Execute with LLM     │       │
        │    │    - Tool calls allowed │       │
        │    │    - Stream response    │       │
        │    └───────────┬─────────────┘       │
        │                │                     │
        │                ▼                     │
        │    ┌─────────────────────────┐       │
        │    │ 5. Store Result         │       │
        │    │    taskResults[id] =    │       │
        │    │      result_text        │       │
        │    └─────────────────────────┘       │
        │                                      │
        ▼                                      ▼
    [Result stored for dependent tasks]   [All tasks complete]
                                               │
                                               ▼
                              ┌────────────────────────────┐
                              │ Step 4: Final Summary      │
                              │ ┌────────────────────────┐ │
                              │ │ Build Summary Context: │ │
                              │ │ ---                    │ │
                              │ │ 原始用户请求: [msg]     │ │
                              │ │ 各关键子任务执行结果:    │ │
                              │ │ - task1: [result]      │ │
                              │ │ - task2: [result]      │ │
                              │ │ ...                    │ │
                              │ └───────────┬────────────┘ │
                              │             │              │
                              │             ▼              │
                              │ ┌────────────────────────┐ │
                              │ │ Full Summary Instruction│ │
                              │ │ ---                    │ │
                              │ │ [summary context]      │ │
                              │ │                        │ │
                              │ │ 请根据以上所有子任务的   │ │
                              │ │ 执行结果，完成以下汇总   │ │
                              │ │ 任务:                  │ │
                              │ │ [final_summary_instr.] │ │
                              │ │                        │ │
                              │ │ 请提供一个完整、连贯的   │ │
                              │ │ 最终回答。              │ │
                              │ └───────────┬────────────┘ │
                              │             │              │
                              │             ▼              │
                              │ ┌────────────────────────┐ │
                              │ │ Regular Agent (Not     │ │
                              │ │ Subtask - Full System  │ │
                              │ │ Prompt & Personality)  │ │
                              │ └───────────┬────────────┘ │
                              └─────────────┼──────────────┘
                                            │
                                            ▼
                              ┌────────────────────────────┐
                              │  Stream Final Response     │
                              │  to User                   │
                              └────────────────────────────┘
```

### Data Flow

**Input:**
- User message describing complex task
- Chat history
- Workspace path
- Token limits and thresholds

**Phase 1 - Planning:**
1. User message → Plan Generation Prompt
2. LLM analyzes and decomposes task
3. Returns JSON with tasks array and dependencies
4. System parses JSON and builds ExecutionGraph

**Phase 2 - Parallel Execution:**
1. For each task without dependencies:
   - Build context from original message
   - Build context from dependency results (if any)
   - Combine with task instruction
   - Create subtask agent with restricted prompt
   - Execute and stream intermediate results
   - Store result in taskResults map
2. Tasks with dependencies wait for prerequisite completion
3. Once prerequisites complete, dependent tasks execute

**Phase 3 - Summary:**
1. Identify leaf tasks (tasks not depended upon)
2. Build summary context with leaf task results
3. Combine with final_summary_instruction from plan
4. Execute with regular agent (not subtask agent)
5. Stream final comprehensive answer to user

**Output:**
- Streamed progress logs for each task
- Intermediate results from subtasks
- Final synthesized answer
- All task results stored for potential follow-up

### Key Differences from Standard Pipeline

1. **Subtask Agent Prompt:** Uses `SUBTASK_AGENT_PROMPT_TEMPLATE` instead of regular system prompt
   - No personality or memory
   - Cannot use `wait_for_user_need` status
   - Strict parallel tool calling requirement
   - Must summarize findings, not output raw data

2. **Context Building:** Each subtask receives:
   - Original user request (for overall context)
   - Current task name
   - Results from dependent tasks
   - Specific task instruction

3. **Summary Agent:** Final summary uses regular system prompt
   - Full personality
   - Access to all features
   - Can wait for user input if needed

---

