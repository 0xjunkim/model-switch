# model-switch

An OpenClaw skill for safely switching AI models and authentication profiles — born from hours of painful debugging.

## Why this exists

Setting up OpenClaw model configuration through the built-in `openclaw configure --section model` interactive picker is broken. We hit every possible failure mode:

1. **Model name corruption** — The interactive picker injects UI rendering artifacts (e.g. `│  ◻ amazon-bedrock/global.anthropic.claude-sonnet-4-5-20250929-v1:0`) into the config file as the model name, causing silent 401 failures.
2. **Token truncation** — OAuth tokens copied from the terminal get split by line breaks. A token that should be 90+ characters ends up as 65, and the API rejects it with no useful error.
3. **Type-field mismatch** — `type: "api_key"` paired with a `token:` field (or vice versa) passes no validation but fails every API call.
4. **Cooldown death spiral** — After auth failures, OpenClaw's cooldown mechanism (`cooldownUntil`, `failureCounts`) blocks retries even after the root cause is fixed. Without manual reset, the agent stays locked out.

All four of these happened in a single setup session. This skill encodes the fixes so no one has to debug them again.

## What it does

When triggered (e.g. "모델 바꿔줘", "switch model"), the skill instructs the OpenClaw agent to:

1. **Read current state** — Parse `openclaw.json` and `auth-profiles.json`
2. **Present options** — Show available models (sonnet/opus/haiku) and auth profiles (OAuth/API key) with masked credentials
3. **Validate tokens** — Check length (≥80 chars), prefix (`sk-ant-oat01-` / `sk-ant-api03-`), type-field consistency, and cooldown status
4. **Edit config directly** — Bypass the buggy interactive picker entirely; write correct model IDs straight to the config file
5. **Reset cooldowns** — Clear `cooldownUntil`, `failureCounts`, `errorCount` in `usageStats`
6. **Restart & verify** — `openclaw gateway restart` + health check

## Security constraints

- Token/key values are **never printed** in messages. Masked format only: `sk-ant-***...***QAA`
- Test script uses only mock data (`FAKE_KEY_FOR_TESTING`, `FAKE_TRUNCATED`)
- No real credentials are stored in this repository

## Files

| File | Purpose |
|------|---------|
| `SKILL.md` | Skill definition — step-by-step procedure for the OpenClaw agent |
| `test-skill.sh` | Validation script — 9 tests covering all failure modes |
| `README.md` | This file |

## Test results

```
✅ PASS: 현재 모델 정상 읽기
✅ PASS: 프로필 3개 정상 로드, 마스킹 적용
✅ PASS: 잘린 토큰 탐지 성공
✅ PASS: 접두사 검증 로직 정상 동작
✅ PASS: type-field 불일치 탐지 성공
✅ PASS: 쿨다운 탐지 로직 정상
✅ PASS: 모델 전환 성공 (opus → sonnet)
✅ PASS: 쿨다운 초기화 + lastGood 변경 성공
✅ PASS: 토큰 마스킹 정상 (전체 값 미노출)
```

Run tests locally:

```bash
bash test-skill.sh
```

## Usage

Place `SKILL.md` in your OpenClaw workspace skills directory:

```
~/.openclaw/workspace/skills/model-switch/SKILL.md
```

Then ask your agent: "switch model" or "모델 바꿔줘"

## Supported models

| Alias | Model ID |
|-------|----------|
| sonnet | `anthropic/claude-sonnet-4-5` |
| opus | `anthropic/claude-opus-4-5` |
| haiku | `anthropic/claude-haiku-3-5` |

## License

MIT
