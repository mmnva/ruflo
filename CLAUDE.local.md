# Local Development Configuration

## Environment Variables

```bash
CLAUDE_FLOW_CONFIG=./claude-flow.config.json
CLAUDE_FLOW_LOG_LEVEL=info
CLAUDE_FLOW_MEMORY_BACKEND=hybrid
CLAUDE_FLOW_MEMORY_PATH=./data/memory
CLAUDE_FLOW_DB_PATH=./data/memory/db  # Three-tier: --flag > env var > default
CLAUDE_FLOW_MCP_PORT=3000
CLAUDE_FLOW_MCP_TRANSPORT=stdio
```

### CLI Flag Precedence Rule
When adding a new CLAUDE_FLOW_* env var:
- CLI flags MUST take precedence (explicit > implicit)
- Document: `// --flag > ENV_VAR > default`
- CI audit enforces this (fails closed)
Example: `const path = args.dbPath || process.env.CLAUDE_FLOW_DB_PATH || './data/memory/db'`

## Plugin Registry Maintenance (IPFS/Pinata)

Registry CID stored in: `v3/@claude-flow/cli/src/plugins/store/discovery.ts`
Gateway: `https://gateway.pinata.cloud/ipfs/{CID}`

Steps to add a plugin:
1. Fetch current registry: `curl -s "https://gateway.pinata.cloud/ipfs/$(grep LIVE_REGISTRY_CID v3/@claude-flow/cli/src/plugins/store/discovery.ts | cut -d"'" -f2)" > /tmp/registry.json`
2. Add plugin entry to `plugins` array, increment `totalPlugins`, update category counts
3. Upload: `curl -X POST "https://api.pinata.cloud/pinning/pinJSONToIPFS" -H "Authorization: Bearer $PINATA_JWT" -H "Content-Type: application/json" -d @/tmp/registry.json`
4. Update `LIVE_REGISTRY_CID` in discovery.ts and the `demoPluginRegistry` fallback

Security: NEVER hardcode API keys. Source from .env at runtime. NEVER commit .env.

## CI Audit Guards (NEW — 2026-05-26)

Three new strict audits block PRs if violations found:

1. **audit-umbrella-version-lockstep.mjs** (#2151/#2153)
   - All 3 packages (@claude-flow/cli, claude-flow, ruflo) must be identical version
   - Ruflo's dep range must include cli's actual version
   
2. **audit-env-var-precedence.mjs** (#2144/ADR-125)
   - Scans CLAUDE_FLOW_* reads for missing CLI-flag precedence documentation
   - Prevents env var from silently ignoring explicit --flags
   
3. **audit-plugin-hooks-cross-platform.mjs** (#2132)
   - Blocks /bin/bash literals, POSIX pipes (jq, xargs, tr), .sh invocations
   - Catches exit 126 failures on Windows before shipping

## Doctor Health Checks

`npx claude-flow@v3alpha doctor` checks: Node 20+, npm 9+, git, config, daemon, memory DB, API keys, MCP servers, disk space, TypeScript.

## Hooks Quick Reference

```bash
npx claude-flow@v3alpha hooks pre-task --description "[task]"
npx claude-flow@v3alpha hooks post-task --task-id "[id]" --success true
npx claude-flow@v3alpha hooks session-start --session-id "[id]"
npx claude-flow@v3alpha hooks route --task "[task]"
npx claude-flow@v3alpha hooks worker list
```

## Intelligence System (RuVector)

4-step pipeline: RETRIEVE (HNSW) → JUDGE (verdicts) → DISTILL (LoRA) → CONSOLIDATE (EWC++)

Components: SONA (<0.05ms), MoE (8 experts), HNSW (150x-12,500x), Flash Attention (2.49x-7.47x)
