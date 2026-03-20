---
layout: post
title: "OpenCode, Agents & Skills"
date: 2026-03-20
tags:
  - OpenCode
  - Agents
  - AI
  - Development
---

In the rapidly evolving landscape of AI-assisted software development, **OpenCode** has emerged as a significant player, particularly with its focus on **agents** and **skills**. This post delves into what these components are, how they function within the OpenCode ecosystem, and why they matter for developers looking to leverage AI more effectively.

What is OpenCode?
Opencode is an open-source AI agent designed to help plan, implement, debug, and refactor entire codebases autonomously. Unlike traditional coding assistants that only autocomplete code or generate passive snippets in a chat window, OpenCode operates primarily through a continuous "agentic loop." This means it can actively explore your file system, execute terminal commands, and modify code independently to achieve a complex goal. 

Opencode supports a wide variety of state-of-the-art AI models—from proprietary giants like Claude, OpenAI, and Gemini, to powerful locally-hosted models. Through its modular architecture of "Agents" and "Skills," developers can seamlessly create and share highly specialized workflows that are perfectly tailored to their specific tech stacks and organizational standards.

Install Opencode - https://opencode.ai/

In **OpenCode**, agents are specialized AI assistants configured to handle specific tasks, workflows, and operations throughout the code  development lifecycle. Instead of using a single monolithic AI for everything, OpenCode allows developers to define and use multiple targeted agents that possess custom prompts, utilize different underlying AI models, and have explicitly defined tool access.

Here is a breakdown of how agents in OpenCode work:

## 1. Types of Agents
- **Primary Agents**: These are the main assistants you interact with directly in your workflow. Standard examples include the **"Build"** agent (a full-access AI for writing, modifying, and executing code) and the **"Plan"** agent (a read-only AI used for code analysis, architecture design, and exploration).
- **Subagents**: These are specialized "worker" assistants that primary agents can seamlessly invoke when they encounter a specific or complex sub-task (e.g., a subagent dedicated purely to advanced web searching or analyzing specific logs).

## 2. Tools and Automation (The Agentic Loop)
OpenCode agents operate on an **"agentic loop."** This means they don't just predict text—they autonomously analyze a goal, decide which tools to use, execute actions, read the results, and iterate until the task is complete. Their standard toolset includes:
- **File Operations**: Reading, writing, and precisely modifying logic in your workspace.
- **Terminal Execution**: Running bash commands, scripts, and tests.
- **LSP Diagnostics**: Checking for syntax and linting errors using the Language Server Protocol.
- **Web Fetching**: Pulling in external context, APIs, or documentation from the internet.

## 3. Agent Configuration
Agents are highly customizable and can be configured using an `opencode.json` file or Markdown specs. You can control:
- Their specific **role** and system prompt.
- Allowed **tools** (e.g., restricting an agent from running bash commands to keep it safely read-only).
- The **AI Model** they use (OpenCode is provider-agnostic and prevents vendor lock-in, supporting Claude, OpenAI, Gemini, or locally hosted open-source models).
- The **Temperature** (controlling how creative or deterministic the agent's output is).

An OpenCode custom agent is essentially a Markdown file (.md) containing a YAML front matter contaning metadata such as name, description, mode (primary/sub), model, temperature, and tool access (e.g., bash: false, write: true) and instructions.

example of an agent in opencode:
```yaml
(base)  ~/.opencode/agents/ cat coding-review.md
---
id: coding-review
name: Coding Review
description: "Multi-language review agent for modular and functional coding"
category: development
type: standard
version: 1.0.0
author: opencode
mode: primary
temperature: 0.1
tools:
  read: true
  edit: false
  write: false
---

# Code Review Agent
Always start with phrase "CODING REVIEW..."

Focus:
You are a coding specialist focused on writing clean, maintainable, and scalable code. Your role is to review code using modular and functional programming principles.

Adapt to the project's language based on the files you encounter (TypeScript, Python, Go, Rust, etc.).

Core Responsibilities
Implement applications with focus on:

- Modular architecture design
- check for potential bugs & edge case
- Clean code principles
- Security consideration
```

I ran the above Coding Review agent on a python project using one of Opencode free model and it gave me the following output:

<img  src="{{ site.baseurl }}/img/opencode-coding-review.png">


## 4. Collaboration and "Skills"
Agents in OpenCode can collaborate with one another, handing off tasks when specialized knowledge is needed. Furthermore, OpenCode incorporates **Agentic Skills**—reusable instructions and capability modules that you can attach to an agent. This teaches the agent exactly how to handle specialized frameworks, specific codebases, or distinct organizational patterns without having to rewrite custom prompts every time.

I will be exploring more on skills.

In short, OpenCode agents act as a modular, autonomous engineering team living within your terminal or IDE, each precisely tailored to perform specific development duties efficiently.


