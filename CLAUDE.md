# OpenFang — Agent Instructions

## Project Overview
OpenFang is an open-source Agent Operating System written in Rust (14 crates).
- Config: `~/.openfang/config.toml`
- Default API: `http://127.0.0.1:4200`
- CLI binary: `target/release/openfang.exe` (or `target/debug/openfang.exe`)

## Build & Verify Workflow
After every feature implementation, run ALL THREE checks:
```bash
cargo build --workspace --lib          # Must compile (use --lib if exe is locked)
cargo test --workspace                 # All tests must pass (currently 1744+)
cargo clippy --workspace --all-targets -- -D warnings  # Zero warnings
```

## MANDATORY: Live Integration Testing
**After implementing any new endpoint, feature, or wiring change, you MUST run live integration tests.** Unit tests alone are not enough — they can pass while the feature is actually dead code. Live tests catch:
- Missing route registrations in server.rs
- Config fields not being deserialized from TOML
- Type mismatches between kernel and API layers
- Endpoints that compile but return wrong/empty data

### How to Run Live Integration Tests

#### Step 1: Stop any running daemon
```bash
tasklist | grep -i openfang
taskkill //PID <pid> //F
# Wait 2-3 seconds for port to release
sleep 3
```

#### Step 2: Build fresh release binary
```bash
cargo build --release -p openfang-cli
```

#### Step 3: Start daemon with required API keys
```bash
GROQ_API_KEY=<key> target/release/openfang.exe start &
sleep 6  # Wait for full boot
curl -s http://127.0.0.1:4200/api/health  # Verify it's up
```
The daemon command is `start` (not `daemon`).

#### Step 4: Test every new endpoint
```bash
# GET endpoints — verify they return real data, not empty/null
curl -s http://127.0.0.1:4200/api/<new-endpoint>

# POST/PUT endpoints — send real payloads
curl -s -X POST http://127.0.0.1:4200/api/<endpoint> \
  -H "Content-Type: application/json" \
  -d '{"field": "value"}'

# Verify write endpoints persist — read back after writing
curl -s -X PUT http://127.0.0.1:4200/api/<endpoint> -d '...'
curl -s http://127.0.0.1:4200/api/<endpoint>  # Should reflect the update
```

#### Step 5: Test real LLM integration
```bash
# Get an agent ID
curl -s http://127.0.0.1:4200/api/agents | python3 -c "import sys,json; print(json.load(sys.stdin)[0]['id'])"

# Send a real message (triggers actual LLM call to Groq/OpenAI)
curl -s -X POST "http://127.0.0.1:4200/api/agents/<id>/message" \
  -H "Content-Type: application/json" \
  -d '{"message": "Say hello in 5 words."}'
```

#### Step 6: Verify side effects
After an LLM call, verify that any metering/cost/usage tracking updated:
```bash
curl -s http://127.0.0.1:4200/api/budget       # Cost should have increased
curl -s http://127.0.0.1:4200/api/budget/agents  # Per-agent spend should show
```

#### Step 7: Verify dashboard HTML
```bash
# Check that new UI components exist in the served HTML
curl -s http://127.0.0.1:4200/ | grep -c "newComponentName"
# Should return > 0
```

#### Step 8: Cleanup
```bash
tasklist | grep -i openfang
taskkill //PID <pid> //F
```

### Key API Endpoints for Testing
| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/health` | GET | Basic health check |
| `/api/agents` | GET | List all agents |
| `/api/agents/{id}/message` | POST | Send message (triggers LLM) |
| `/api/budget` | GET/PUT | Global budget status/update |
| `/api/budget/agents` | GET | Per-agent cost ranking |
| `/api/budget/agents/{id}` | GET | Single agent budget detail |
| `/api/network/status` | GET | OFP network status |
| `/api/peers` | GET | Connected OFP peers |
| `/api/a2a/agents` | GET | External A2A agents |
| `/api/a2a/discover` | POST | Discover A2A agent at URL |
| `/api/a2a/send` | POST | Send task to external A2A agent |
| `/api/a2a/tasks/{id}/status` | GET | Check external A2A task status |

## Raspberry Pi 400 Cross-Build (GitHub Actions)
The project targets Raspberry Pi 400 (aarch64). Windows cannot build natively (OpenSSL/Perl issues).
- Workflow: `.github/workflows/build-rpi.yml` (manual `workflow_dispatch`)
- Target: `aarch64-unknown-linux-gnu`
- Uses `cross` for cross-compilation
- Artifact: `openfang-aarch64-unknown-linux-gnu.tar.gz`
- To trigger: Go to GitHub Actions → "Build for Raspberry Pi (aarch64)" → Run workflow

## Extra Headers Feature (for Coding Plan / DashScope)
`config.toml` supports `[default_model.extra_headers]` to send custom HTTP headers with LLM requests.
This is used for Alibaba Cloud Coding Plan which requires specific User-Agent headers.

```toml
[default_model]
provider = "openai"
model = "qwen3.5-plus"
api_key_env = "QWEN_API_KEY"
base_url = "https://coding-intl.dashscope.aliyuncs.com/v1"

[default_model.extra_headers]
User-Agent = "OpenClaw-Gateway/1.0"
```

### Files involved in extra_headers
- `DriverConfig` struct: `crates/openfang-runtime/src/llm_driver.rs` — `extra_headers: Vec<(String, String)>`
- `DefaultModelConfig`: `crates/openfang-types/src/config.rs` — `extra_headers: HashMap<String, String>`
- `ModelConfig` (agent-level): `crates/openfang-types/src/agent.rs` — `extra_headers: HashMap<String, String>`
- Kernel wiring: `crates/openfang-kernel/src/kernel.rs` — converts HashMap→Vec when building DriverConfig
- Driver factory: `crates/openfang-runtime/src/drivers/mod.rs` — applies via `OpenAIDriver::with_extra_headers()`
- OpenAI driver: `crates/openfang-runtime/src/drivers/openai.rs` — `with_extra_headers()` already existed

## Architecture Notes
- **Don't touch `openfang-cli`** — user is actively building the interactive CLI
- `KernelHandle` trait avoids circular deps between runtime and kernel
- `AppState` in `server.rs` bridges kernel to API routes
- New routes must be registered in `server.rs` router AND implemented in `routes.rs`
- Dashboard is Alpine.js SPA in `static/index_body.html` — new tabs need both HTML and JS data/methods
- Config fields need: struct field + `#[serde(default)]` + Default impl entry + Serialize/Deserialize derives

## Current Status (2026-03-08)

### extra_headers ブランチ: `feat/extra-headers`
- **コード変更**: 完了・コミット済み (`8428087`)
- **プッシュ**: `origin/feat/extra-headers` にプッシュ済み
- **main マージ**: まだ（PR未作成）
- **ビルド検証**: Windows では OpenSSL/Perl 問題のためビルド不可。GitHub Actions `build-rpi.yml` で検証が必要
- **Cargo.lock**: `main` に別途バージョンバンプ (0.3.25→0.3.26) + `openssl-src` 追加の変更あり（未ステージ）

### 残りの手順
1. `feat/extra-headers` を `main` にマージ（PR or 直接マージ）
2. GitHub Actions → "Build for Raspberry Pi (aarch64)" → Run workflow（main ブランチ指定）
3. ビルド成功後、アーティファクトをダウンロード → ラズパイにデプロイ
4. ラズパイの `config.toml` に以下を設定:
   ```toml
   [default_model.extra_headers]
   User-Agent = "OpenClaw-Gateway/1.0"
   ```
5. デーモン再起動して動作確認:
   ```bash
   curl -X POST http://127.0.0.1:4200/v1/chat/completions \
     -H "Content-Type: application/json" \
     -d '{"model":"qwen3.5-plus","messages":[{"role":"user","content":"Hello"}]}'
   ```

## Common Gotchas
- `openfang.exe` may be locked if daemon is running — use `--lib` flag or kill daemon first
- `PeerRegistry` is `Option<PeerRegistry>` on kernel but `Option<Arc<PeerRegistry>>` on `AppState` — wrap with `.as_ref().map(|r| Arc::new(r.clone()))`
- Config fields added to `KernelConfig` struct MUST also be added to the `Default` impl or build fails
- `AgentLoopResult` field is `.response` not `.response_text`
- CLI command to start daemon is `start` not `daemon`
- On Windows: use `taskkill //PID <pid> //F` (double slashes in MSYS2/Git Bash)
- Windows cannot build the full binary (OpenSSL/Perl issue) — use GitHub Actions `build-rpi.yml` for aarch64 builds
- Every `DriverConfig { ... }` construction site must include `extra_headers` field (currently 7 sites across kernel.rs, routes.rs, drivers/mod.rs)
