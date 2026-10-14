# Koog AI Agent Framework

Koog is a Kotlin multiplatform framework for building AI agents with graph-based workflows.
It supports JVM and JS targets and integrates with multiple LLM providers
(OpenAI, Anthropic, Google, OpenRouter, Ollama) and Model Context Protocol (MCP).

## Solufee Fork 工作流

本仓库是 [JetBrains/koog](https://github.com/JetBrains/koog) 的 fork（`solufee/koog`）。

- 操作 GitHub 前先执行 `gh auth switch --user solufee`。

### 分支

| 分支 | 用途 |
|------|------|
| `develop` | 始终与上游 `upstream/develop` 保持一致，**禁止**提交任何 solufee 自有改动 |
| `solufee-develop` | 上游 + solufee 补丁，是我们的主开发分支 |
| `solufee/feat/<name>`、`solufee/fix/<name>` | 单个补丁分支，从 `solufee-develop` 切出 |

- 远程：`origin` = `solufee/koog`，`upstream` = `JetBrains/koog`。
- 同步上游：`git fetch upstream`，将 `develop` 快进到 `upstream/develop` 并推送到 `origin`，再把 `develop` 合入 `solufee-develop`。
- 每个补丁分支都以 **squash merge** 合并到 `solufee-develop`，一个补丁对应 `solufee-develop` 上的一个提交。

### 版本与发布

- Maven 产物发布到 GitHub Packages（`https://maven.pkg.github.com/solufee/koog`）。
- 版本号 = 上游 koog 版本 + `-solufee.<number>`，例如 `2.0.1-solufee.1`。
- `<number>` 在同一上游版本上的每次发布递增；上游 koog 版本更新后重置为 `1`。
- 上游版本是 `gradle.properties` 中的 `version`，`<number>` 是同文件中的 `solufee.number`。
- 只有加 `-Psolufee.release=true` 时才使用 solufee 版本号，否则仍是上游的 `-SNAPSHOT` 逻辑。
- 发布需要带 `write:packages` 权限的 token。Gradle 自带的 8.x 不支持 JDK 25，请用 JDK 17/21：

```bash
JAVA_HOME=<jdk-21> \
ORG_GRADLE_PROJECT_gpr.user=solufee \
ORG_GRADLE_PROJECT_gpr.key="$(gh auth token -u solufee)" \
./gradlew -Psolufee.release=true publishAllPublicationsToGitHubPackagesRepository
```

## Project Structure

The project follows a modular architecture with a clear separation of concerns:

```
koog/
├── agents/
│   ├── agents-core/           # Core abstractions (AIAgent, AIAgentStrategy, AIAgentEnvironment)
│   ├── agents-tools/          # Tool infrastructure (Tool<TArgs, TResult>, ToolRegistry, AIAgentTool)
│   ├── agents-features-*/     # Feature implementations (memory, tracing, event handling)
│   ├── agents-mcp/           # Model Context Protocol integration
│   └── agents-test/          # Testing utilities and framework
├── prompt-*/                 # LLM interaction layer (executors, models, structured data)
├── embeddings-*/             # Vector embedding support
├── examples/                 # Reference implementations and usage patterns
└── build.gradle.kts          # Root build configuration
```

## Build & Commands

### Development Commands

```bash
# Full build including tests
./gradlew build

# Build without tests
./gradlew assemble

# Run all JVM tests
./gradlew jvmTest

# Run all JS tests  
./gradlew jsTest

# Test specific module
./gradlew :agents:agents-core:jvmTest

# Run specific test class
./gradlew jvmTest --tests "ai.koog.agents.test.SimpleAgentMockedTest"

# Run specific test method  
./gradlew jvmTest --tests "ai.koog.agents.test.SimpleAgentMockedTest.test AIAgent doesn't call tools by default"

# Compile test classes only (for faster iteration)
./gradlew jvmTestClasses jsTestClasses
```

### Development Environment

- **JDK**: 17+ required for JVM target
- **Build System**: Gradle with version catalogs for dependency management
- **Targets**: JVM, JavaScript (Kotlin Multiplatform), WASM
- **IDE**: IntelliJ IDEA recommended with Kotlin plugin

## Code Style

- Follow [Kotlin Coding Conventions](https://kotlinlang.org/docs/coding-conventions.html)
- Use four spaces for indentation (consistent across all files)
- Name test functions as `testXxx` (no backticks for readability)
- Use descriptive variable and function names
- Prefer functional programming patterns where appropriate
- Use type-safe builders and DSLs for configuration
- Document public APIs with KDoc comments
- NEVER suppress compiler warnings without a good reason

## Quality Gates
Read and follow the Quality Gates section in /TESTING.md before considering any code change complete.

## Architecture

### Core Framework Components

**AIAgent** — Main orchestrator that executes strategies in coroutine scopes, manages tools via ToolRegistry,
runs features through AIAgentPipeline, and handles LLM communication via PromptExecutor.

**AIAgentStrategy** — Graph-based execution logic that defines workflows as subgraphs with start/finish nodes,
manages tool selection strategy, and handles termination/error reporting.

**ToolRegistry** — Centralized, type-safe tool management using a builder pattern: `ToolRegistry { tool(MyTool()) }`.
Supports registry merging with `+` operator.

**AIAgentFeature** — Extensible capabilities installed into AIAgentPipeline with configuration.
Features have unique storage keys and can intercept agent lifecycle events.

### Module Organization

1. **agents-core**: Core abstractions (`AIAgent`, `AIAgentStrategy`, `AIAgentEnvironment`)
2. **agents-tools**: Tool infrastructure (`Tool<TArgs, TResult>`, `ToolRegistry`, `AIAgentTool`)
3. **agents-features-***: Feature implementations (memory, tracing, event handling)
4. **agents-mcp**: Model Context Protocol integration
5. **prompt-***: LLM interaction layer (executors, models, structured data)
6. **embeddings-***: Vector embedding support
7. **examples**: Reference implementations and usage patterns

### Key Architectural Patterns

- **State Machine Graphs**: Agents execute as node graphs with typed edges
- **Feature Pipeline**: Extensible behavior via installable features with lifecycle hooks
- **Environment Abstraction**: Safe tool execution context preventing direct tool calls
- **Type Safety**: Generics ensure compile-time correctness for tool arguments/results
- **Builder Patterns**: Fluent APIs for configuration throughout the framework

## Testing

The framework provides comprehensive testing utilities in `agents-test` module:

### LLM Response Mocking

```kotlin
val mockLLMApi = getMockExecutor(toolRegistry, eventHandler) {
    mockLLMAnswer("Hello!") onRequestContains "Hello"
    mockLLMToolCall(CreateTool, CreateTool.Args("solve")) onRequestEquals "Solve task"
    mockLLMAnswer("Default response").asDefaultResponse
}
```

### Tool Behavior Mocking

```kotlin
// Simple return value
mockTool(PositiveToneTool) alwaysReturns "The text has a positive tone."

// With additional actions
mockTool(NegativeToneTool) alwaysTells {
    println("Tool called")
    "The text has a negative tone."
}

// Conditional responses
mockTool(SearchTool) returns SearchTool.Result("Found") onArgumentsMatching {
    args.query.contains("important")
}
```

### Graph Structure Testing

```kotlin
AIAgent(...) {
    withTesting()

    testGraph("test") {
        val firstSubgraph = assertSubgraphByName<String, String>("first")
        val secondSubgraph = assertSubgraphByName<String, String>("second")

        assertEdges {
            startNode() alwaysGoesTo firstSubgraph
            firstSubgraph alwaysGoesTo secondSubgraph
        }

        verifySubgraph(firstSubgraph) {
            val askLLM = assertNodeByName<String, Message.Response>("callLLM")
            assertNodes {
                askLLM withInput "Hello" outputs Message.Assistant("Hello!")
            }
        }
    }
}
```

For comprehensive testing examples, see `agents/agents-test/TESTING.md`.

## Security

### API Key Management

- **NEVER** commit API keys or secrets to the repository
- Use environment variables for all sensitive configuration
- Store test API keys in a local environment only
- Required environment variables for integration tests:
    - `ANTHROPIC_API_TEST_KEY`
    - `GEMINI_API_TEST_KEY`
    - `MISTRAL_AI_API_TEST_KEY`
    - `OLLAMA_IMAGE_URL`
    - `OPEN_AI_API_TEST_KEY`
    - `OPEN_ROUTER_API_TEST_KEY`

### Tool Execution Safety

- Tools execute within controlled `AIAgentEnvironment` contexts
- Direct tool calls are prevented outside agent execution
- Use type-safe tool arguments to prevent injection attacks
- Validate all external inputs in tool implementations

### Dependency Security

- Regularly update dependencies using Gradle version catalogs
- Use specific version ranges to avoid supply chain attacks
- Review dependencies for known vulnerabilities
- Follow the principle of the least privilege in tool implementations

## Configuration

### Environment Setup

Set environment variables for integration testing (never commit API keys):

```bash
# Export in your shell or IDE run configuration
export ANTHROPIC_API_TEST_KEY=your_key_here
export DEEPSEEK_API_TEST_KEY=your_key_here
export GEMINI_API_TEST_KEY=your_key_here
export MISTRAL_AI_API_TEST_KEY=your_key_here
export OLLAMA_IMAGE_URL=http://localhost:11434
export OPEN_AI_API_TEST_KEY=your_key_here
export OPEN_ROUTER_API_TEST_KEY=your_key_here

# Or add to ~/.bashrc, ~/.zshrc, or IDE environment variables
```

### Gradle Configuration

- Uses version catalogs (`gradle/libs.versions.toml`) for dependency management
- Multiplatform configuration in `build.gradle.kts`
- Test configuration supports both JVM and JS targets

### Development Environment Requirements

- **JDK**: 17+ (OpenJDK recommended)
- **IDE**: IntelliJ IDEA with Kotlin Multiplatform plugin
- **Optional**: Docker for Ollama local testing

## Development Workflow

### Branch Strategy

> 以下是上游 JetBrains 的分支策略。在 solufee fork 中开发时，以上文「Solufee Fork 工作流」为准。

- **develop**: All development (features and bug fixes)
- **main**: Released versions only
- Base all PRs against `develop` branch
- Use descriptive branch names: `feature/agent-memory`, `fix/tool-registry-bug`

### Code Quality

- **ALWAYS** run `./gradlew build` before submitting PRs
- Ensure all tests pass on JVM, JS, WASM targets
- Follow established patterns in existing code
- Add tests for new functionality
- Update documentation for API changes

### Commit Guidelines

- Use conventional commit format: `feat:`, `fix:`, `docs:`, `test:`
- Include issue references where applicable
- Keep commits focused and atomic
