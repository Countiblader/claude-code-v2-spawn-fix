# Claude Code v2 — SDK Spawn Fix

Quick migration note for anyone building tools that spawn the Claude Code CLI
(`@anthropic-ai/claude-code`) programmatically. With **Claude Code 2.x**,
Anthropic replaced the JavaScript entry point (`cli.js`) with a **native
binary** (`bin/claude.exe` on Windows). Integrations that hardcoded
`node node_modules/@anthropic-ai/claude-code/cli.js` started crashing
with `MODULE_NOT_FOUND` on user machines after the update.

## The Symptom

```
Error: Cannot find module '...node_modules/@anthropic-ai/claude-code/cli.js'
    at resolveForCJSWithHooks (node:internal/modules/cjs/loader:1064:22)
    at Module._load (node:internal/modules/cjs/loader:1234:25)
    ...
  code: 'MODULE_NOT_FOUND',
  requireStack: []
```

The empty `requireStack: []` is the giveaway — Node.js cannot find the
**entry file itself** (not a `require()` inside it).

## Why It Broke

Old layout (v1.x):
```
node_modules/@anthropic-ai/claude-code/
├── cli.js           ← Node entry, spawnable via `node cli.js`
└── package.json
```

New layout (v2.x, tested on 2.1.114):
```
node_modules/@anthropic-ai/claude-code/
├── bin/
│   └── claude.exe   ← native binary (Windows), no more JS entry
├── cli-wrapper.cjs  ← fallback, only if postinstall skipped
├── sdk-tools.d.ts
└── package.json     ← "bin": { "claude": "bin/claude.exe" }
```

There is **no `cli.js`** anymore. The old spawn path is dead.

## The Fix

Spawn the installed shim (`claude.cmd` on Windows, `claude` elsewhere)
**directly**. It's what npm creates in your `PATH` for exactly this reason.

### TypeScript / Node.js

```ts
// ❌ Old (broken on v2.x)
const cliJs = path.join(cliDir, "node_modules/@anthropic-ai/claude-code/cli.js");
spawn("node", [cliJs, ...args]);

// ✅ New
// If you already know the full path to claude.cmd / claude, spawn it directly.
// No "node" prefix — the shim handles the Node bootstrap (or invokes the native binary).
spawn(cliPath, args);
```

If you only have the command name (`"claude"`), let the OS resolve it from
`PATH`. On Windows you may need `{ shell: true }` for `.cmd` resolution when
using `child_process.spawn`.

```ts
spawn("claude", args, { shell: process.platform === "win32" });
```

### Rust (tokio / std)

```rust
use tokio::process::Command;

// Windows: cli_path ends with claude.cmd or claude.exe
// macOS/Linux: cli_path is the full path to the `claude` shim
let mut cmd = Command::new(&cli_path);
cmd.args(&args);
cmd.spawn()?;
```

### Detecting the path

Windows:
```bash
where claude
# e.g. C:\Users\YOU\AppData\Roaming\npm\claude.cmd
```

macOS / Linux:
```bash
which claude
# e.g. /usr/local/bin/claude  (or ~/.npm-global/bin/claude, ~/.volta/bin/claude, etc.)
```

If `where` / `which` misses it on Windows (nvm, volta, fnm, custom PATH),
fall back to PowerShell:
```powershell
(Get-Command claude).Source
```

## Related Anthropic Issues

- [anthropics/claude-code#10191 — sdk.mjs entry point missing](https://github.com/anthropics/claude-code/issues/10191)
- [anthropics/claude-code#7238 — SDK unable to find executable](https://github.com/anthropics/claude-code/issues/7238)
- [anthropics/claude-code#3838 — claude CLI not recognized after global install on Windows](https://github.com/anthropics/claude-code/issues/3838)
- [anthropics/claude-code#43547 — CLI 2.1.92 hardcoded CI-build paths](https://github.com/anthropics/claude-code/issues/43547)

## Notes

- `claude.cmd` is a standard npm-generated shell wrapper on Windows — spawn it
  directly, no `node` prefix.
- The native binary lives at
  `node_modules/@anthropic-ai/claude-code/bin/claude.exe`. You can spawn it
  directly too if you resolve that path.
- `cli-wrapper.cjs` in the package is only a fallback for when postinstall was
  skipped (`npm install --ignore-scripts`). Prefer the shim / native binary.
- If your integration uses the official `@anthropic-ai/claude-code` SDK
  module, pass an explicit `pathToClaudeCodeExecutable` option — auto-resolve
  still looks for the retired `cli.js` in some versions (see issue #7238).

## License

MIT. Use and adapt freely.

## Credits

Diagnosis and fix written collaboratively with [Claude Code](https://claude.com/claude-code)
while debugging the same bug in a production SDK integration. Published so
others hitting the same `MODULE_NOT_FOUND` can skip the root-cause hunt.
