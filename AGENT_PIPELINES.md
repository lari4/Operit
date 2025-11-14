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

## 3. Memory & Knowledge Graph Pipeline

**Purpose:** Automatically extracts knowledge from conversations and builds a persistent knowledge graph with entities, relationships, and user preferences.

**Entry Point:** After task completion (when `<status type="complete">` is detected)

**Key Components:**
- `ProblemLibrary`
- `MemoryRepository`
- `GraphService`
- Knowledge graph analysis AI agent

### ASCII Flow Diagram

```
┌─────────────────────────────┐
│  Task Completion Detected   │
│  <status type="complete">   │
└──────────┬──────────────────┘
           │
           ▼
┌──────────────────────────────────────────┐
│  Check if Auto-Analysis Enabled          │
└──────────┬───────────────────────────────┘
           │ (Yes)
           ▼
┌──────────────────────────────────────────┐
│  ProblemLibrary.analyzeConversation()    │
│  ┌────────────────────────────────────┐  │
│  │ Collect Inputs:                    │  │
│  │ - Recent conversation (query)      │  │
│  │ - AI's solution                    │  │
│  │ - Conversation history             │  │
│  │ - Current user preferences         │  │
│  └─────────────┬──────────────────────┘  │
└────────────────┼─────────────────────────┘
                 │
                 ▼
┌──────────────────────────────────────────┐
│  Query Similar Memories                  │
│  - Use hybrid search (keyword + semantic)│
│  - Retrieve existing relevant memories   │
│  - Find existing folder structure        │
└──────────┬───────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────┐
│  Detect Duplicates                       │
│  - Find memories with same title         │
│  - Prepare merge instructions            │
└──────────┬───────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────────────┐
│  Build Knowledge Graph Analysis Prompt               │
│  ┌────────────────────────────────────────────────┐  │
│  │ System Prompt Components:                      │  │
│  │ - Base: Knowledge graph expert role            │  │
│  │ - Duplicates to merge (if any)                 │  │
│  │ - Existing memories context                    │  │
│  │ - Existing folder structure                    │  │
│  │ - Memory filtering principles                  │  │
│  │   * Prioritize: user info, unique concepts     │  │
│  │   * Avoid: common knowledge, famous facts      │  │
│  │ - Memory attributes (credibility, importance)  │  │
│  │ - Output format (compact JSON)                 │  │
│  │ - Current user preferences                     │  │
│  └─────────────┬──────────────────────────────────┘  │
│                │                                      │
│                ▼                                      │
│  ┌────────────────────────────────────────────────┐  │
│  │ User Message:                                  │  │
│  │ 问题: [query]                                   │  │
│  │ 解决方案: [solution, truncated to 3000 chars]  │  │
│  │ 历史记录: [last 10 turns]                       │  │
│  └─────────────┬──────────────────────────────────┘  │
└────────────────┼─────────────────────────────────────┘
                 │
                 ▼
┌──────────────────────────────────────────┐
│  Send to Analysis Agent                  │
│  - Dedicated AI service                  │
│  - Processes conversation                │
└──────────┬───────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────────────┐
│  AI Returns Compact JSON                             │
│  {                                                    │
│    "main": ["title", "content", ["tags"], "folder"], │
│    "new": [                                           │
│      ["entity", "content", ["tags"], "folder", null] │
│    ],                                                 │
│    "update": [                                        │
│      ["title", "new_content", "reason", 0.9, 0.8]    │
│    ],                                                 │
│    "merge": [                                         │
│      {                                                │
│        "source_titles": ["title1", "title2"],        │
│        "new_title": "merged",                        │
│        "new_content": "...",                         │
│        "reason": "..."                               │
│      }                                                │
│    ],                                                 │
│    "links": [                                         │
│      ["source", "target", "IS_A", "desc", 1.0]       │
│    ],                                                 │
│    "user": {                                          │
│      "personality": "...",                           │
│      "occupation": "<UNCHANGED>"                     │
│    }                                                  │
│  }                                                    │
└──────────┬───────────────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────┐
│  Parse & Validate JSON                   │
└──────────┬───────────────────────────────┘
           │
           ├──────────────────┬──────────────┬─────────────┬─────────────┐
           │                  │              │             │             │
           ▼                  ▼              ▼             ▼             ▼
    ┌──────────┐      ┌───────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
    │  Main    │      │    New    │  │  Update  │  │  Merge   │  │  Links   │
    │ Knowledge│      │ Entities  │  │ Existing │  │  Dupes   │  │          │
    └────┬─────┘      └─────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘
         │                  │              │             │             │
         ▼                  ▼              ▼             ▼             ▼
┌────────────────────────────────────────────────────────────────────────────┐
│  Memory Repository Operations                                             │
│  - Create main knowledge node                                             │
│  - Create new entity nodes (check aliases)                                │
│  - Update existing memories (content, credibility, importance)            │
│  - Merge duplicate memories                                               │
│  - Create graph edges with weights                                        │
└──────────┬─────────────────────────────────────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────┐
│  Update User Preferences                 │
│  - Only update changed fields            │
│  - Preserve "<UNCHANGED>" fields         │
│  - Save to PreferenceProfile             │
└──────────┬───────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────┐
│  Knowledge Graph Updated                 │
│  - New memories indexed                  │
│  - Embeddings generated                  │
│  - Graph relationships established       │
│  - Ready for future queries              │
└──────────────────────────────────────────┘
```

### Data Flow

**Input:**
- Conversation query (user's question/request)
- AI solution (complete response)
- Conversation history (last 10 turns)
- Current user preferences
- Existing relevant memories
- Existing folder structure

**Processing:**
1. Query similar memories (hybrid search)
2. Detect duplicate memories by title
3. Build comprehensive analysis prompt with all context
4. AI analyzes and extracts:
   - Core knowledge (main node)
   - Individual entities (new nodes)
   - Updates to existing memories
   - Merges for duplicates
   - Relationships between entities
   - Updated user preferences
5. Parse compact JSON response
6. Execute database operations:
   - Create/update memory nodes
   - Merge duplicates
   - Create graph edges
   - Update user profile

**Output:**
- New memory nodes in database
- Updated existing memories
- Graph relationships with weights
- Updated user preference profile
- Embeddings for semantic search

### Memory Quality Control

**Filtering Principles:**
- **Record:** User-specific info, unique concepts, file contents, post-cutoff events
- **Avoid:** Common knowledge, famous facts, publicly available information

**Attributes:**
- `credibility` (0.0-1.0): Affects content representation in retrieval
- `importance` (0.0-1.0): Affects search ranking weight
- `edge.weight` (0.0-1.0): Relationship strength

**Deduplication Strategy:**
- Use `alias_for` field to link duplicate entities
- Merge operation combines multiple similar memories
- Prioritize updating over creating new memories

---

## 4. Conversation Summarization Pipeline

**Purpose:** Creates comprehensive summaries of conversations to maintain context in long sessions without exceeding token limits.

**Entry Point:** Triggered automatically or manually when conversation becomes long

**Key Components:**
- `ConversationService`
- Summary AI agent
- Conversation storage

### ASCII Flow Diagram

```
┌─────────────────────────────┐
│  Summarization Triggered    │
│  (Auto or Manual)           │
└──────────┬──────────────────┘
           │
           ▼
┌──────────────────────────────────────────┐
│  Collect Summary Inputs                  │
│  - Previous summary (if exists)          │
│  - Recent conversation turns             │
└──────────┬───────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────────────┐
│  Build Summary Prompt                                │
│  ┌────────────────────────────────────────────────┐  │
│  │ System: SUMMARY_PROMPT                         │  │
│  │                                                │  │
│  │ Requirements:                                  │  │
│  │ - Fixed format with sections                  │  │
│  │ - 【核心任务状态】                              │  │
│  │   * Last user request                         │  │
│  │   * Current progress step                     │  │
│  │   * Completion status                         │  │
│  │ - 【对话历程与概要】                            │  │
│  │   * Evolution of conversation                 │  │
│  │   * Key turning points                        │  │
│  │   * How requirements were handled             │  │
│  │ - 【关键信息与上下文】                          │  │
│  │   * Specific requirements                     │  │
│  │   * File names, code snippets                 │  │
│  │   * Technical details                         │  │
│  │   * Solutions attempted                       │  │
│  └─────────────┬──────────────────────────────────┘  │
│                │                                      │
│                ▼                                      │
│  ┌────────────────────────────────────────────────┐  │
│  │ User Message:                                  │  │
│  │ 上一次的摘要: [previous summary]                │  │
│  │ 最近的对话内容: [recent turns]                  │  │
│  └────────────────────────────────────────────────┘  │
└──────────┬───────────────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────┐
│  Send to Summary Agent                   │
└──────────┬───────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────────────┐
│  AI Returns Structured Summary                       │
│  ==========对话摘要==========                          │
│                                                       │
│  【核心任务状态】                                       │
│  - 最后一条用户需求: ...                               │
│  - 当前进行到哪一步: ...                               │
│  - 任务是否完成: 进行中/已完成/等待中                    │
│                                                       │
│  【对话历程与概要】                                     │
│  [Detailed multi-paragraph overview]                 │
│                                                       │
│  【关键信息与上下文】                                   │
│  - [Key detail 1]                                    │
│  - [Key detail 2]                                    │
│  ...                                                 │
│                                                       │
│  ============================                        │
└──────────┬───────────────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────┐
│  Store Summary                           │
│  - Replace old chat history              │
│  - Summary becomes new context           │
└──────────┬───────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────┐
│  Continue Conversation                   │
│  - Next messages include summary         │
│  - No need to send full history          │
└──────────────────────────────────────────┘
```

### Data Flow

**Input:**
- Previous summary (if exists)
- Recent conversation content

**Processing:**
1. Build structured summary prompt
2. AI analyzes and creates self-contained summary
3. Summary must be complete enough to replace entire history

**Output:**
- Comprehensive structured summary
- Replaces conversation history in next interactions

**Key Principles:**
- Professional, clear, objective language
- No length limit - completeness over brevity
- Must be self-contained (AI can understand context from summary alone)

---

## 5. UI Automation Pipeline

**Purpose:** Enables AI to control Android UI through analyzing screenshots and executing actions.

**Entry Point:** User requests UI automation task

**Key Components:**
- `UIControllerService`
- Vision-capable AI model
- UI accessibility service

### ASCII Flow Diagram

```
┌─────────────────────────────┐
│  User Task Goal             │
│  "Open Settings and enable  │
│   dark mode"                │
└──────────┬──────────────────┘
           │
           ▼
┌──────────────────────────────────────────┐
│  Initialize UI Automation                │
│  - Capture current UI state              │
│  - Start execution history log           │
└──────────┬───────────────────────────────┘
           │
           ▼
        ┌──────────────────────┐
        │  Automation Loop     │
        └──────┬───────────────┘
               │
       ┌───────▼────────┐
       │  Current Step  │
       └───────┬────────┘
               │
               ▼
┌──────────────────────────────────────────────────────┐
│  Capture UI State                                    │
│  - Screenshot (if vision model available)            │
│  - UI element tree (accessibility)                   │
│  - Element properties (bounds, text, clickable, etc) │
└──────────┬───────────────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────────────┐
│  Build UI Controller Prompt                         │
│  ┌────────────────────────────────────────────────┐  │
│  │ System: UI_CONTROLLER_PROMPT                   │  │
│  │                                                │  │
│  │ Inputs:                                        │  │
│  │ 1. Current UI State                            │  │
│  │    - List of elements with properties          │  │
│  │ 2. Task Goal                                   │  │
│  │    - Specific objective for this step          │  │
│  │ 3. Execution History                           │  │
│  │    - Previous actions and outcomes             │  │
│  │                                                │  │
│  │ Output Format: Single JSON                     │  │
│  │ {                                              │  │
│  │   "explanation": "...",                        │  │
│  │   "command": {                                 │  │
│  │     "type": "action_type",                     │  │
│  │     "arg": {...}                               │  │
│  │   }                                            │  │
│  │ }                                              │  │
│  └────────────────────────────────────────────────┘  │
└──────────┬───────────────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────┐
│  Send to UI Controller Agent             │
│  - Vision model if screenshot available  │
│  - Text-only if accessibility tree only  │
└──────────┬───────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────────────┐
│  AI Returns Action JSON                              │
│  {                                                    │
│    "explanation": "Tapping Settings icon...",        │
│    "command": {                                       │
│      "type": "tap",                                   │
│      "arg": {"x": 540, "y": 960}                     │
│    }                                                  │
│  }                                                    │
└──────────┬───────────────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────┐
│  Parse JSON & Execute Command            │
│                                          │
│  Action Types:                           │
│  - tap: Click at coordinates             │
│  - swipe: Swipe gesture                  │
│  - set_input_text: Type text             │
│  - press_key: Press keycode              │
│  - start_app: Launch app by package      │
│  - list_installed_apps: Query apps       │
│  - complete: Task finished               │
│  - interrupt: Task failed/blocked        │
└──────────┬───────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────┐
│  Log Action Result                       │
│  - Add to execution history              │
│  - Record explanation & outcome          │
└──────────┬───────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────┐
│  Check Command Type                      │
└──────────┬───────────────────────────────┘
           │
           ├───────────┬──────────────┐
           │           │              │
   ┌───────▼───┐  ┌───▼───────┐  ┌──▼──────────┐
   │ complete  │  │ interrupt │  │  UI Action  │
   └───────┬───┘  └───────┬───┘  └──┬──────────┘
           │              │          │
           ▼              ▼          │
   ┌─────────────┐  ┌────────────┐  │
   │ Task Done   │  │ Task Failed│  │
   └─────────────┘  └────────────┘  │
                                    │
                    ┌───────────────┘
                    │
                    ▼
            ┌───────────────┐
            │ Wait & Loop   │
            │ Back to Top   │
            └───────────────┘
```

### Data Flow

**Input:**
- Task goal (user's objective)
- UI state (screenshot + accessibility tree)
- Execution history (previous actions)

**Processing Loop:**
1. Capture current UI state
2. Build prompt with UI elements, goal, and history
3. AI analyzes and decides next action
4. Parse JSON command
5. Execute action on UI
6. Log result in history
7. If not complete/interrupted, repeat

**Output:**
- Sequence of UI actions
- Execution log
- Task completion or failure status

**Available Actions:**
- **UI Interaction:** tap, swipe, set_input_text, press_key
- **App Management:** start_app, list_installed_apps
- **Control:** complete, interrupt

---

## 6. Voice Mode Pipeline

**Purpose:** Optimized conversational flow for text-to-speech output with concise, natural responses.

**Entry Point:** User switches to voice mode

**Key Differences from Standard Chat:**

### System Prompt Modifications

```
┌─────────────────────────────────────────┐
│  Voice Mode System Prompt               │
│                                         │
│  Base Components (Same as Chat):       │
│  - Behavior Guidelines                  │
│  - Tool Usage Guidelines                │
│  ✓ Thinking Guidance (Disabled)         │
│  ✓ Memory Query (Disabled)              │
│                                         │
│  Voice-Specific Components:             │
│  ┌───────────────────────────────────┐  │
│  │ Default Voice Intro:              │  │
│  │ "你是Operit语音助手。你的所有回答  │  │
│  │  都将通过语音播出，所以你必须只说  │  │
│  │  那些听起来自然的话。"            │  │
│  └───────────────────────────────────┘  │
│  ┌───────────────────────────────────┐  │
│  │ Voice Tone Guidelines:            │  │
│  │ - 回答必须非常简短、口语化         │  │
│  │ - 像日常聊天一样                  │  │
│  │ - 严禁使用列表、分点              │  │
│  │ - 严禁使用Markdown标记            │  │
│  │ - 纯文本、可直接朗读的对话         │  │
│  │ - 直接回答，不要客套话            │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
```

### Response Processing

```
User Message
     │
     ▼
AI Response (Concise, Conversational)
     │
     ▼
Text-to-Speech Engine
     │
     ▼
Audio Output
```

**Key Constraints:**
- No lists or bullet points
- No Markdown formatting
- Short, natural sentences
- Direct answers only
- Tools can still be used but responses must remain conversational

---

## 7. Translation Pipeline

**Purpose:** Simple translation service preserving tone and style.

### ASCII Flow Diagram

```
┌─────────────────┐
│  Text to        │
│  Translate      │
└────────┬────────┘
         │
         ▼
┌──────────────────────────────────────────┐
│  Build Translation Prompt                │
│  ┌────────────────────────────────────┐  │
│  │ System:                            │  │
│  │ "你是一个专业的翻译助手，能够准确   │  │
│  │  翻译各种语言，并保持原文的语气和   │  │
│  │  风格。"                           │  │
│  └────────────────────────────────────┘  │
│  ┌────────────────────────────────────┐  │
│  │ User:                              │  │
│  │ "请将以下文本翻译为[language]，     │  │
│  │  保持原文的语气和风格：            │  │
│  │                                    │  │
│  │  [text]                            │  │
│  │                                    │  │
│  │  只返回翻译结果，不要添加任何解释   │  │
│  │  或额外内容。"                     │  │
│  └────────────────────────────────────┘  │
└──────────┬───────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────┐
│  Translation AI Agent                    │
└──────────┬───────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────┐
│  Return Translated Text Only             │
└──────────────────────────────────────────┘
```

**Supported Target Languages:**
- 中文 (Chinese) - Default
- English

---

## 8. Package Description Generation Pipeline

**Purpose:** Auto-generates user-friendly descriptions for MCP tool packages.

### ASCII Flow Diagram

```
┌─────────────────────────────┐
│  New MCP Package Added      │
│  - Package name             │
│  - Tool descriptions list   │
└──────────┬──────────────────┘
           │
           ▼
┌──────────────────────────────────────────────────────┐
│  Build Package Description Prompt                   │
│  ┌────────────────────────────────────────────────┐  │
│  │ User Message:                                  │  │
│  │ "请为名为'[name]'的MCP工具包生成一个简洁的描述。│  │
│  │  这个工具包包含以下工具：                       │  │
│  │                                                │  │
│  │  [tool list]                                   │  │
│  │                                                │  │
│  │  要求：                                        │  │
│  │  1. 描述应该简洁明了，不超过100字               │  │
│  │  2. 重点说明工具包的主要功能和用途              │  │
│  │  3. 使用中文                                   │  │
│  │  4. 不要包含技术细节，要通俗易懂                │  │
│  │  5. 只返回描述内容，不要添加任何其他文字"        │  │
│  └────────────────────────────────────────────────┘  │
└──────────┬───────────────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────┐
│  Description Generation Agent            │
└──────────┬───────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────┐
│  Return Concise Description              │
│  - ≤100 characters                       │
│  - User-friendly language                │
│  - Main functionality highlighted        │
└──────────┬───────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────┐
│  Save to Package Metadata               │
└──────────────────────────────────────────┘
```

---

## Summary of Agent Types

Operit uses several specialized agent types with different prompts and capabilities:

| Agent Type | Prompt Template | Tools | Memory | Personality | Wait for User | Use Case |
|------------|----------------|-------|--------|-------------|---------------|----------|
| **Standard Chat** | System Prompt + Custom | ✓ | ✓ | ✓ | ✓ | General conversation |
| **Subtask (Plan Mode)** | Subtask Agent Template | ✓ | ✗ | ✗ | ✗ | Parallel task execution |
| **Summary** | Summary Prompt | ✗ | ✗ | ✗ | ✗ | Conversation summarization |
| **Final Summary (Plan)** | System Prompt + Custom | ✓ | ✓ | ✓ | ✓ | Plan mode result synthesis |
| **Knowledge Graph** | KG Analysis Prompt | ✗ | ✗ | ✗ | ✗ | Memory extraction |
| **UI Controller** | UI Controller Prompt | ✗ | ✗ | ✗ | ✗ | UI automation |
| **Voice Mode** | Voice System Prompt | ✓ | ✗ | ✓ | ✓ | Voice interactions |
| **Translation** | Translation Prompt | ✗ | ✗ | ✗ | ✗ | Text translation |
| **Package Describer** | Package Desc. Prompt | ✗ | ✗ | ✗ | ✗ | MCP package descriptions |

---

## Data Flow Between Systems

### Complete Conversation Lifecycle

```
User Message
    │
    ▼
┌─────────────────────────────────────┐
│  Conversation Service               │
│  - Load preferences                 │
│  - Build system prompt              │
│  - Apply personality                │
└─────────────┬───────────────────────┘
              │
              ▼
┌─────────────────────────────────────┐
│  Enhanced AI Service                │
│  - Route to correct provider        │
│  - Stream response                  │
└─────────────┬───────────────────────┘
              │
              ▼
    ┌─────────┴─────────┐
    │                   │
    ▼                   ▼
┌────────┐      ┌──────────────┐
│  Text  │      │  Tool Call   │
└───┬────┘      └──────┬───────┘
    │                  │
    │                  ▼
    │          ┌──────────────────┐
    │          │  Execute Tool    │
    │          └──────┬───────────┘
    │                 │
    │                 ▼
    │          ┌──────────────────┐
    │          │  Next Turn       │
    │          └──────────────────┘
    │
    ▼
┌─────────────────────────────────────┐
│  Task Complete                      │
└─────────────┬───────────────────────┘
              │
              ├──────────────┬─────────────────┐
              │              │                 │
              ▼              ▼                 ▼
    ┌─────────────┐  ┌──────────────┐  ┌─────────────┐
    │ Summarize   │  │ Update       │  │ User sees   │
    │ (if needed) │  │ Knowledge    │  │ response    │
    │             │  │ Graph        │  │             │
    └──────┬──────┘  └──────┬───────┘  └─────────────┘
           │                │
           │                ▼
           │         ┌──────────────────┐
           │         │ Memory Repository│
           │         │ - New nodes      │
           │         │ - Relationships  │
           │         │ - User prefs     │
           │         └──────────────────┘
           │
           ▼
    ┌──────────────────┐
    │ Summary Storage  │
    │ - Replace history│
    └──────────────────┘
```

This comprehensive pipeline system enables Operit to handle a wide variety of tasks while maintaining context, learning from conversations, and adapting to user preferences.

