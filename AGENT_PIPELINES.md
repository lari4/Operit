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

