# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Litho** (package name: `deepwiki-rs`) is a high-performance AI-driven documentation generation engine built with Rust. It automatically analyzes codebases and generates comprehensive C4 architecture documentation through a sophisticated multi-stage processing pipeline with specialized AI agents.

## Quick Start with Claude Code

If you're using Claude Code to work on this project, here's how to get started:

### 1. Set up your development environment
```bash
# Make sure Rust is installed
rustup --version

# Build the project
cargo build

# Run tests to verify setup
cargo test
```

### 2. Configure LLM for testing Litho
You'll need an LLM API key to run Litho. Pick one:

**Option A: DeepSeek (Cost-Effective)**
```bash
export LITHO_LLM_API_KEY="sk-your-deepseek-key"
cargo run -- -p ./src --llm-provider deepseek --model-efficient deepseek-chat
```

**Option B: OpenAI**
```bash
export LITHO_LLM_API_KEY="sk-proj-your-openai-key"
cargo run -- -p ./src --model-efficient gpt-4o-mini
```

**Option C: Ollama (Free, Local)**
```bash
ollama pull llama3.1
cargo run -- -p ./src --llm-provider ollama --model-efficient llama3.1
```

### 3. Understand the configuration system
- Configuration file: `litho.toml.example` (copy to `litho.toml`)
- See **Configuration** section below for details
- Vietnamese users: Read `SETUP_VI.md` for detailed setup guide

### 4. Making changes
Before making changes, understand:
- The four-stage pipeline (see **High-Level Architecture**)
- Agent pattern (see **AI Agent Architecture**)
- Memory system (see **Memory System**)
- How models are used in each stage (see **Model Usage by Pipeline Stage**)

## Development Commands

### Building and Running
```bash
# Build the project
cargo build --release

# Build for development
cargo build

# Run with default settings (analyzes current directory)
cargo run

# Run with specific project path
cargo run -- -p ./path/to/project -o ./output

# Install from source
cargo install --path .
```

### Testing
```bash
# Run all tests
cargo test

# Run tests with output
cargo test -- --nocapture

# Run specific test
cargo test <test_name>
```

### Code Quality
```bash
# Check code without building
cargo check

# Format code
cargo fmt

# Run linter
cargo clippy
```

## High-Level Architecture

### Four-Stage Processing Pipeline

Litho implements a deterministic pipeline with four core stages:

1. **Preprocessing** (`src/generator/preprocess/`) - Scans codebase, extracts structure, parses multiple languages, and initializes agent memory chunks
2. **Research & Analysis** (`src/generator/research/`) - AI-powered stage using specialized agents (System Context, Domain Module Detection, Workflow Analysis, Boundary Analysis, Key Module Insights) with ReAct reasoning loops
3. **Documentation Generation** (`src/generator/compose/`) - Generates structured documentation using multiple editor agents (Overview, Architecture, Workflow, Boundary, Module Insights)
4. **Verification & Enhancement** (`src/generator/outlet/`) - Validates Mermaid diagrams, checks integrity, auto-repairs syntax, and outputs final documentation

### Core Modules

- **`src/main.rs`** - Entry point, parses CLI args and launches workflow
- **`src/generator/workflow.rs`** - Orchestrates the entire four-stage pipeline
- **`src/generator/context.rs`** - `GeneratorContext` holds shared state (LLM client, cache, memory)
- **`src/memory/mod.rs`** - Unified memory manager using scope:key pattern for data storage across agents
- **`src/llm/client/`** - LLM integration layer supporting multiple providers (OpenAI, Anthropic, DeepSeek, Mistral, OpenRouter, Gemini, Ollama)
- **`src/cache/`** - Caching system for LLM responses to reduce API calls
- **`src/config.rs`** - Configuration management (loads from `litho.toml` or CLI args)
- **`src/cli.rs`** - Command-line argument parsing using clap

### Language Support

Multi-language parsers in `src/generator/preprocess/extractors/language_processors/`:
- Rust, Python, Java, JavaScript, TypeScript
- Kotlin, Go, C#
- Frontend frameworks: React, Vue, Svelte

### AI Agent Architecture

Agents follow the **StepForwardAgent** pattern (`src/generator/step_forward_agent.rs`):
- Each agent validates data sources from memory
- Executes via `AgentExecutor` with prompt or extraction logic
- Results cached via `CacheManager`
- Stores outputs back to memory for downstream agents

Research agents (`src/generator/research/agents/`):
- `system_context_researcher.rs` - Analyzes overall system context
- `domain_modules_detector.rs` - Identifies domain boundaries
- `architecture_researcher.rs` - Infers architectural patterns
- `workflow_researcher.rs` - Maps data flows and processes
- `key_modules_insight.rs` - Deep-dives into critical modules

Composition agents (`src/generator/compose/agents/`):
- `overview_editor.rs` - Generates project overview
- `architecture_editor.rs` - Creates architecture diagrams
- `workflow_editor.rs` - Documents workflows
- `boundary_editor.rs` - Documents boundaries
- `key_modules_insight_editor.rs` - Detailed module documentation

## Configuration

### Config File (litho.toml)

Place `litho.toml` in project root for persistent configuration. Config precedence: CLI args > `litho.toml` > defaults.

### LLM Provider Configuration

Litho supports two model tiers with different purposes:

| Model Tier | Parameter | Purpose | Used In | Example Models |
|------------|-----------|---------|---------|----------------|
| **Efficient** | `--model-efficient` | Fast, cost-effective model for routine analysis | Preprocessing, basic research tasks | gpt-4o-mini, claude-haiku, gemini-flash, deepseek-chat |
| **Powerful** | `--model-powerful` | High-quality model for complex reasoning | Complex research, documentation generation | gpt-4o, claude-sonnet, gemini-pro |

**Default Configuration** (from `src/config.rs:473-490`):
- Provider: ModelScope (Chinese provider)
- Base URL: `https://api-inference.modelscope.cn/v1`
- Efficient Model: `Qwen/Qwen3-Next-80B-A3B-Instruct`
- Powerful Model: `Qwen/Qwen3-235B-A22B-Instruct-2507`
- API Key: Read from `LITHO_LLM_API_KEY` environment variable

**Supported Providers**: `openai`, `anthropic`, `deepseek`, `mistral`, `openrouter`, `gemini`, `ollama`

### Model Usage by Pipeline Stage

Understanding which models are used where helps optimize cost and performance:

1. **Preprocessing Stage** → Primarily `model_efficient`
   - Code scanning and structure extraction
   - Batch processing many files
   - Cost-sensitive operations

2. **Research Stage** → Mix of both models
   - SystemContextResearcher → `model_efficient`
   - DomainModulesDetector → `model_efficient`
   - ArchitectureResearcher → `model_powerful`
   - WorkflowResearcher → `model_powerful`
   - BoundaryAnalyzer → `model_efficient`
   - KeyModulesInsight → `model_powerful`

3. **Documentation Generation** → Primarily `model_powerful`
   - All editor agents use powerful model for quality
   - Critical for generating professional output

4. **Verification Stage** → No LLM calls
   - Syntax validation and auto-repair

### Provider-Specific Examples

#### OpenAI (Recommended for Quality)
```bash
cargo run -- -p ./src \
  --llm-provider openai \
  --llm-api-base-url https://api.openai.com/v1 \
  --llm-api-key sk-proj-xxx \
  --model-efficient gpt-4o-mini \
  --model-powerful gpt-4o
```

#### Anthropic Claude (Best for Code Analysis)
```bash
cargo run -- -p ./src \
  --llm-provider anthropic \
  --llm-api-base-url https://api.anthropic.com \
  --llm-api-key sk-ant-xxx \
  --model-efficient claude-3-haiku-20240307 \
  --model-powerful claude-3-5-sonnet-20241022
```

#### DeepSeek (Cost-Effective)
```bash
cargo run -- -p ./src \
  --llm-provider deepseek \
  --llm-api-base-url https://api.deepseek.com/v1 \
  --llm-api-key sk-xxx \
  --model-efficient deepseek-chat \
  --model-powerful deepseek-chat
```

#### Google Gemini (Lowest Cost)
```bash
cargo run -- -p ./src \
  --llm-provider gemini \
  --llm-api-base-url https://generativelanguage.googleapis.com/v1beta \
  --llm-api-key AIzaSy-xxx \
  --model-efficient gemini-1.5-flash \
  --model-powerful gemini-1.5-pro
```

#### Ollama (Free, Local, Private)
```bash
# First: ollama pull llama3.1
cargo run -- -p ./src \
  --llm-provider ollama \
  --llm-api-base-url http://localhost:11434 \
  --model-efficient llama3.1 \
  --model-powerful llama3.1
```

### Cost Comparison (Approximate, January 2025)

| Provider | Efficient Model | Cost/1M tokens | Powerful Model | Cost/1M tokens |
|----------|-----------------|----------------|----------------|----------------|
| **Gemini** | gemini-1.5-flash | $0.075 | gemini-1.5-pro | $1.25 |
| **DeepSeek** | deepseek-chat | $0.14 | deepseek-chat | $0.14 |
| **OpenAI** | gpt-4o-mini | $0.15 | gpt-4o | $2.50 |
| **Anthropic** | claude-haiku | $0.25 | claude-sonnet | $3.00 |
| **Ollama** | llama3.1 | FREE | llama3.1 | FREE |

**Estimated costs for 5000-file project (~2M tokens)**:
- Gemini: ~$0.50
- DeepSeek: ~$0.30
- OpenAI (mixed): ~$1.50
- Claude: ~$3.50
- Ollama: $0 (requires GPU)

### Important CLI Flags

```bash
# Skip pipeline stages (useful for iterative development)
--skip-preprocessing    # Skip code scanning phase
--skip-research        # Skip AI analysis phase
--skip-documentation   # Skip doc generation phase

# Disable ReAct mode (no auto-scanning via tool calls)
--disable-preset-tools

# Cache management
--no-cache            # Disable caching
--force-regenerate    # Clear cache and regenerate

# Internationalization
--target-language <lang>  # Output language (en, zh, ja, ko, de, fr, ru)

# Model configuration
--model-efficient <model>
--model-powerful <model>
--llm-provider <provider>
--llm-api-base-url <url>
--llm-api-key <key>
```

## Memory System

The `Memory` struct (`src/memory/mod.rs`) uses a **scope:key** pattern for hierarchical data storage:

```rust
memory.store("preprocess", "project_structure", structure)?;
let structure = memory.get::<ProjectStructure>("preprocess", "project_structure");
```

Common scopes:
- `preprocess` - Project structure, code insights, relationships
- `research` - Research agent outputs
- `compose` - Generated documentation sections
- `timing` - Performance metrics

## Important Implementation Details

### Agent Execution Flow

1. **Preprocessing** extracts raw data → stores in memory
2. **Research orchestrator** runs multiple agents in parallel
3. Each agent reads from memory, calls LLM (with caching), stores results
4. **Documentation composer** reads research outputs, generates structured docs
5. **Outlet** validates and writes to disk

### ReAct Mode

When enabled (default), agents use tool-calling to scan project files dynamically. Tools available in `src/llm/tools/`:
- `file_explorer.rs` - Browse directory structure
- `file_reader.rs` - Read file contents
- `time.rs` - Get current timestamp

Disable with `--disable-preset-tools` for simpler prompt-based generation.

### Cache System

Located at `.litho/cache/` - MD5 hashes of prompts map to cached LLM responses. Use `--force-regenerate` to clear.

### Output Structure

Generated docs saved to `./litho.docs/` (configurable with `-o`):
```
litho.docs/
├── 1. Project Overview.md
├── 2. Architecture Overview.md
├── 3. Workflow Overview.md
├── 4. Deep Dive/
│   ├── <topic1>.md
│   ├── <topic2>.md
└── __Litho_Summary_*.md
```

## Development Workflow

### Adding a New Language Processor

1. Create parser in `src/generator/preprocess/extractors/language_processors/<lang>.rs`
2. Implement structure extraction, comment parsing, dependency detection
3. Register in `language_processors/mod.rs`

### Adding a New Research Agent

1. Create agent in `src/generator/research/agents/<agent_name>.rs`
2. Implement `StepForwardAgent` pattern
3. Define memory scope/keys in agent file
4. Register in `research/orchestrator.rs`

### Modifying Documentation Templates

Agents use embedded prompts. To modify output format:
- Edit prompt templates in respective agent files
- Templates reference i18n (`src/i18n.rs`) for multilingual support

## Edge Cases and Gotchas

- **edition = "2024"** in Cargo.toml - Ensure Rust toolchain supports this edition
- Config file must be valid TOML - malformed `litho.toml` will fall back to defaults with warning
- LLM API keys should be set via environment variables or CLI, never committed
- Cache can grow large - periodically clear `.litho/` directory
- Parallel agent execution requires sufficient API rate limits
