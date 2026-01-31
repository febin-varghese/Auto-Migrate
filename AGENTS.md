# Auto-Type Agent Guidelines

This document provides guidance for agentic coding assistants working on the Auto-Type codebase.

## Project Overview

Auto-Type is an autonomous CLI tool that migrates JavaScript codebases to TypeScript using a ReAct (Reason + Act) loop. The system validates correctness through two-stage validation: TypeScript Compiler (`tsc`) for syntax errors and existing test suites (`npm test`) for logic regressions. It self-corrects until both validation gates pass.

## Build, Lint, and Test Commands

```bash
npm run build          # Compile TypeScript to JavaScript
npm run build:watch    # Watch mode for development
npm run lint           # Run ESLint on all files
npm run lint:fix       # Auto-fix linting issues
npm run format         # Run Prettier to format code
npm run format:check   # Check code formatting
npm test               # Run all tests
npm test:watch         # Run tests in watch mode
npm test:coverage      # Generate coverage report
npm test -- <file>     # Run specific test file
npm test -t "<name>"   # Run tests matching a name pattern
npm run dev            # Start development server
npm run typecheck      # Run TypeScript type checking only
```

## Code Style Guidelines

### Imports
Use ES6 import syntax. Group imports: external libraries, internal modules (from `src/`), relative paths. Avoid default exports; prefer named exports.
```typescript
import { Command } from 'commander';
import chalk from 'chalk';
import ora from 'ora';
import { Agent } from 'src/core/agent';
import { retry } from './retry';
```

### Formatting
- 2 spaces indentation, semicolons, single quotes, backticks for templates
- Max line length: 100 characters
- Trailing commas in multi-line arrays/objects

### TypeScript Types
No `any` types. Use `interface` for shapes, `type` for unions/aliases. Explicit return types on public functions. Use `unknown` over `any`. Enable strict mode.
```typescript
interface MigrationConfig { sourcePath: string; targetPath: string; }
type ToolResult = { success: true; data: unknown } | { success: false; error: string };
async function executeTool(tool: Tool): Promise<ToolResult> { }
```

### Naming Conventions
- **Files**: kebab-case (`dependency-finder.ts`)
- **Variables/Functions**: camelCase (`collectMetrics`)
- **Classes/Interfaces**: PascalCase (`Agent`, `MigrationConfig`)
- **Constants**: SCREAMING_SNAKE_CASE (`MAX_RETRIES`)
- **Private members**: prefix with underscore (`_internalState`)

### Error Handling
Always handle async errors with try/catch. Create custom error classes extending Error. Use Result type pattern. Never swallow errors silently.
```typescript
class ToolExecutionError extends Error {
  constructor(tool: string, cause: unknown) {
    super(`Tool execution failed: ${tool}`);
    this.cause = cause;
  }
}

async function safeExecute(): Promise<Result> {
  try {
    const result = await riskyOperation();
    return { success: true, value: result };
  } catch (error) {
    logger.error('Operation failed', { error });
    return { success: false, error: error instanceof Error ? error.message : String(error) };
  }
}
```

### Async/Await
Use async/await over Promise chains. Await async operations in series. Use `Promise.all()` for parallel operations. Avoid mixing callbacks.

### Logging
Use centralized logger from `src/utils/logger.ts` (Winston-based). Include context objects. Use appropriate levels: error, warn, info, debug. For CLI: use `ora` spinners and `chalk` colors.
```typescript
logger.info('Starting migration', { sourcePath, targetPath });
logger.error('Tool failed', { tool: 'compiler', error });
logger.debug('Processing file', { filePath, size });
const spinner = ora('Migrating files...').start();
console.log(chalk.green('Migration complete!'));
```

### Testing
Write tests in `.test.ts` or `.spec.ts` files. Use Jest or Vitest. Mock external dependencies (LLM API, file system). Test success and error paths.
```typescript
describe('Agent', () => {
  it('should complete a migration successfully', async () => {
    const agent = new Agent(config);
    const result = await agent.migrate();
    expect(result.success).toBe(true);
  });
});
```

### Tool Development
Tools must follow ReAct pattern: Observe, Think, Act. Single responsibility. Timeout handling for external operations. Return structured results with success/failure status. Use retry utility for transient failures. Compiler tool spawns `child_process.exec` for `npx tsc --noEmit`. Tester tool runs `npm test`, capturing failures for LLM. Max 3 retries per file before marking partial success. On failure: save as `.ts.partial`, log to `errors.json`, continue to next file.

### File Operations
Always use `file-manager.ts`. Validate paths and permissions. Handle encoding explicitly (UTF-8). Use atomic writes (temp → rename).

## Architecture Notes

- **Core**: Main ReAct loop agent implementation with prompts and types
- **Tools**: Process spawning (TSC via `compiler.ts`, npm test via `tester.ts`)
- **Analyzers**: Dependency detection via regex, context window management
- **Evaluators**: Migration metrics (success rate, retries, token usage), reports
- **Utils**: Winston logging and retry logic (max 3 attempts)

### The ReAct Validation Loop
For each file: Generate TS code via LLM → Run `npx tsc --noEmit` → On error: feed stderr to LLM → On success: run `npm test` → On failure: feed test logs → Max 3 retries → Save partial as `.ts.partial`, log to `errors.json`.

### Metrics & Success Criteria
Track these metrics during migration:
- `total_files`: Total files processed
- `successful_migrations`: Files passing Compiler + Tests
- `automation_rate`: % of files needing zero human intervention (target: 85%+)
- `avg_retries`: How many loops per file
- `token_usage`: Cost tracking (target: <$0.10 per file)
Output `migration-report.json` (machine-readable) and `console.table()` summary.

Maintain separation of concerns. Core logic remains framework-agnostic where possible.
