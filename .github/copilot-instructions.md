# TOON Format Codebase - AI Agent Instructions

## Project Overview

TOON (Token-Oriented Object Notation) is a JSON-alternative format optimized for LLM prompts that reduces token usage by 30-60% while maintaining lossless conversion. It combines YAML-like indentation with CSV-style tabular arrays for maximum efficiency.

**Architecture**: TypeScript monorepo with core encoder/decoder library (`packages/toon`), CLI tool (`packages/cli`), and comprehensive benchmarks comparing token efficiency and LLM comprehension across formats.

## Project Structure & Key Patterns

### Monorepo Layout
- `packages/toon/` - Core encoding/decoding library (pure functions, no dependencies)  
- `packages/cli/` - CLI wrapper using `citty` and `consola` for user experience
- `benchmarks/` - Standalone evaluation suite testing token efficiency and LLM accuracy
- Root contains shared config (ESLint, TypeScript, pnpm workspace)

### Core Architecture Patterns

**Encode Pipeline**: `normalize → encode → write`
```typescript
// Entry point validates input, resolves options, delegates to encoder
encode(value) → normalizeValue() → encodeValue() → LineWriter.toString()
```

**Decode Pipeline**: `scan → parse → validate → decode`  
```typescript  
// Parser operates on structured line cursors with validation
decode(input) → toParsedLines() → LineCursor → decodeValueFromLines()
```

**Key folding** (optional): Collapses nested single-key chains into dotted paths (`data.user.name` instead of 3 indent levels) when `keyFolding: 'safe'`. Pairs with `expandPaths: 'safe'` for lossless round-trips.

**Tabular detection**: Arrays of objects with identical primitive fields become CSV-like tables with headers (`items[2]{id,name}: 1,Alice / 2,Bob`). Non-uniform arrays use list format (`items[2]: - {id:1} - text`).

### Critical Implementation Details

**Delimiter awareness**: Parser adapts quoting rules based on active delimiter (`,` default, `\t`, `|`). Strings containing the active delimiter get quoted, others remain unquoted for token efficiency.

**Whitespace significance**: Uses 2-space indentation by default. `LineWriter` tracks depth and manages consistent formatting. Blank lines between sections are preserved during decode.

**Type normalization**: `NaN`/`Infinity` → `null`, `BigInt` → number or quoted string, `Date` → ISO string, `undefined`/`function`/`symbol` → `null`.

## Development Workflows  

### Building & Testing
```bash
pnpm build          # Build all packages (uses tsdown for bundling)
pnpm test           # Run vitest tests across all packages  
pnpm test:types     # TypeScript type checking
pnpm lint           # ESLint with @antfu/eslint-config
```

### Benchmarks (Critical for Performance Validation)
```bash
cd benchmarks
pnpm benchmark:tokens    # Token efficiency vs JSON/YAML/XML/CSV
pnpm benchmark:accuracy  # LLM comprehension tests (requires API keys)
```

**Benchmark architecture**: Uses `@toon-format/spec` conformance tests + generated datasets. Accuracy tests query 4 LLMs across 209 questions (5 types: field retrieval, aggregation, filtering, structure awareness, validation). Results auto-generate markdown reports embedded in main README.

### File Naming Conventions
- `*.test.ts` - Vitest unit tests importing fixtures from `@toon-format/spec`
- `*-benchmark.ts` - Performance evaluation scripts in `benchmarks/scripts/`
- `tsdown.config.ts` - Build configuration for each package
- Shared types in `types.ts`, constants in `constants.ts`

## Key Files for Understanding Core Logic

**Encoder**: `packages/toon/src/encode/encoders.ts` - Main encoding logic with tabular detection
**Decoder**: `packages/toon/src/decode/decoders.ts` - Parser with line cursor navigation  
**Primitives**: `packages/toon/src/encode/primitives.ts` - Quoting rules and value formatting
**Validation**: `packages/toon/src/decode/validation.ts` - Array length and structure validation
**Folding**: `packages/toon/src/encode/folding.ts` - Key path collapse logic

## Testing Strategy

Uses `@toon-format/spec` package for conformance tests ensuring compatibility with the official specification. Tests cover:
- Round-trip fidelity (encode → decode → matches original)
- Edge cases (empty arrays, special characters, unicode)  
- Delimiter variations (comma, tab, pipe)
- Key folding/expansion with collision detection
- Strict vs lenient parsing modes

**Test data pattern**: JSON fixtures with expected TOON output, plus options for delimiter/folding tests.

## Common Development Patterns

**Adding new encoders**: Extend `encodeValue()` switch, add primitive detection in `normalize.ts`, update type guards
**Parser extensions**: Add line pattern recognition in `parser.ts`, implement decoder in `decoders.ts`
**Benchmark datasets**: Add generator in `benchmarks/src/datasets.ts`, questions in `benchmarks/src/questions/`
**CLI options**: Extend types in `packages/cli/src/types.ts`, add argument parsing in CLI entry

## Dependencies & External Integrations

**Core library**: Zero runtime dependencies (pure TypeScript)
**CLI**: `citty` (command framework), `consola` (logging), `tokenx` (token counting)  
**Benchmarks**: `ai` SDK for LLM evaluation, `gpt-tokenizer` for token counting, `faker` for test data
**Build**: `tsdown` (bundling), `vitest` (testing), `@antfu/eslint-config` (linting)

Focus on the encoder/decoder separation and the tabular array detection logic when making changes - these are the core differentiators from other formats.