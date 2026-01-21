# Genie

> *"You ain't never had a friend like me!"* - Genie, Aladdin

Intelligent model router for Claude Code. Give Genie your wish (prompt), and it figures out the best Claude model to grant it.

## Features

- **Smart Model Selection**: Analyzes task complexity and selects optimal model
- **Fast Analysis**: Uses Haiku for quick decisions
- **Spawnable Agent**: Other plugins can use Genie for model selection
- **Brief Explanations**: Shows reasoning without being verbose

## Usage

### Command

```
/genie:wish <your prompt here>
```

Genie will:
1. Analyze your prompt
2. Select the optimal model (haiku/sonnet/opus)
3. Show brief reasoning: `✨ Sonnet - Code implementation task detected`
4. Execute your task with the selected model

### Agent

Other plugins can spawn the Genie agent for model selection:

```
Task(subagent_type="genie:genie", prompt="Select model for: <task description>")
```

## Model Selection Criteria

| Model | Best For |
|-------|----------|
| **Haiku** | Scanning, pattern matching, simple lookups, quick questions |
| **Sonnet** | Code writing, implementation, nuanced reasoning, investigation |
| **Opus** | Complex architecture, design decisions, heavy reasoning |

## Part of the Disney Plugin Collection

Genie lives alongside [Fantasia](../fantasia/) in the wanplugins repository.
