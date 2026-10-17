# Local Development Configuration

## Environment Variables

```bash
CLAUDE_FLOW_CONFIG=./claude-flow.config.json
CLAUDE_FLOW_LOG_LEVEL=info
CLAUDE_FLOW_MEMORY_BACKEND=hybrid
CLAUDE_FLOW_MEMORY_PATH=./data/memory
CLAUDE_FLOW_DB_PATH=./data/memory/memory.db        # NEW: (#2146) Direct memory DB path
CLAUDE_FLOW_MCP_PORT=3000
CLAUDE_FLOW_MCP_TRANSPORT=stdio
```

### CLAUDE_FLOW_DB_PATH Precedence (New in #2146)

When using memory commands (store, retrieve, search, list, delete, stats):
1. `--path` flag (highest priority)
2. `CLAUDE_FLOW_DB_PATH` env var
3. `CLAUDE_FLOW_MEMORY_PATH/memory.db` (fallback)
4. `cwd/.swarm/memory.db` (default root)

## Plugin Registry Maintenance (IPFS/Pinata)

Registry CID stored in: `v3/@claude-flow/cli/src/plugins/store/discovery.ts`
Gateway: `https://gateway.pinata.cloud/ipfs/{CID}`

Steps to add a plugin:
1. Fetch current registry: `curl -s "https://gateway.pinata.cloud/ipfs/$(grep LIVE_REGISTRY_CID v3/@claude-flow/cli/src/plugins/store/discovery.ts | cut -d"'" -f2)" > /tmp/registry.json`
2. Add plugin entry to `plugins` array, increment `totalPlugins`, update category counts
3. Upload: `curl -X POST "https://api.pinata.cloud/pinning/pinJSONToIPFS" -H "Authorization: Bearer $PINATA_JWT" -H "Content-Type: application/json" -d @/tmp/registry.json`
4. Update `LIVE_REGISTRY_CID` in discovery.ts and the `demoPluginRegistry` fallback

Security: NEVER hardcode API keys. Source from .env at runtime. NEVER commit .env.

## Team Gateway Checklist (New in #2146)

See `docs/TEAM-GATEWAY-CHECKLIST.md` for:
- Before-merge gates: lint, type-check, tests, smoke tests, witness manifest, semver
- Dual-mode Claude/Codex handoff protocols
- Memory namespace sharing conventions
- Witness manifest entry requirements per merge
- Gateway logging and audit scope

Key commands for team merges:
```bash
npm run lint && npx tsc --noEmit && npm test
node scripts/smoke-*.mjs                            # All smoke tests
npx ruflo@latest sign --message "merge: <PR> — <description>"
npx ruflo@latest verify                             # Verify dist after install
```

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
