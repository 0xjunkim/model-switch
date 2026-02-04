# Model Switch - 모델 전환 스킬

OpenClaw 모델과 인증 프로필을 안전하게 전환하는 스킬.
트리거: "모델 바꿔줘", "모델 전환", "switch model"

## 보안 제약

- 이 스킬은 **로컬 실행만 허용**. 텔레그램 등 원격 채널에서 호출 시 토큰/키 값은 절대 출력하지 않는다.
- 토큰/키 값을 메시지에 노출하지 않는다. 마스킹 처리: `sk-ant-***...***QAA` 형태로만 표시.
- 파일 수정은 로컬 파일시스템 직접 편집으로만 수행.

## 사용 가능 모델 목록

| 별칭 | 모델 ID | 설명 |
|------|---------|------|
| sonnet | anthropic/claude-sonnet-4-5 | Claude Sonnet 4.5 (비용 효율) |
| opus | anthropic/claude-opus-4-5 | Claude Opus 4.5 (최고 성능) |
| haiku | anthropic/claude-haiku-3-5 | Claude Haiku 3.5 (빠름/저렴) |

> **중요**: `openclaw configure --section model` 의 인터랙티브 picker를 사용하지 않는다.
> picker에서 모델명이 깨져 들어가는 버그가 있음. 항상 직접 config 파일을 편집한다.

## 실행 절차

### 1단계: 현재 상태 확인

다음 두 파일을 읽는다:

1. `~/.openclaw/openclaw.json` → `agents.defaults.model.primary` 확인
2. `~/.openclaw/agents/main/agent/auth-profiles.json` → 현재 활성 프로필, lastGood 확인

사용자에게 현재 상태를 요약:
```
현재 모델: anthropic/claude-sonnet-4-5 (sonnet)
인증 방식: OAuth token (anthropic:default)
상태: 정상
```

### 2단계: 모델 선택

사용자에게 모델 선택지를 제시:
- sonnet (anthropic/claude-sonnet-4-5) - 추천, 비용 효율
- opus (anthropic/claude-opus-4-5) - 고성능
- haiku (anthropic/claude-haiku-3-5) - 빠름

### 3단계: 인증 방식 선택

사용자에게 인증 방식을 확인:
- **OAuth token** (sk-ant-oat01-...): Max 구독 사용자 (추천)
- **API key** (sk-ant-api03-...): API 직접 과금

기존 auth-profiles.json에 등록된 프로필 목록을 보여주되, **키/토큰 값은 마스킹** 처리.

### 4단계: 토큰 유효성 검증

선택된 프로필의 토큰/키를 검증:

1. **길이 체크**: OAuth 토큰(oat01)은 최소 80자 이상이어야 함. 잘린 토큰은 터미널 줄바꿈으로 인해 발생.
2. **접두사 체크**:
   - OAuth: `sk-ant-oat01-` 로 시작
   - API key: `sk-ant-api03-` 로 시작
3. **타입-필드 일치 체크**:
   - `type: "token"` → `token` 필드에 값이 있어야 함
   - `type: "api_key"` → `key` 필드에 값이 있어야 함
4. **쿨다운 상태 체크**: `usageStats`에서 cooldownUntil이 현재 시간 이후이면 경고

문제 발견 시 사용자에게 명확히 안내:
```
⚠ 토큰이 잘려있습니다 (현재 65자, 최소 80자 필요)
  프로필: anthropic:default
  토큰 앞부분: sk-ant-oat01-XXXX...
  → 전체 토큰을 다시 입력해주세요
```

### 5단계: config 파일 수정

`~/.openclaw/openclaw.json` 수정:

```json
{
  "agents": {
    "defaults": {
      "model": {
        "primary": "anthropic/claude-sonnet-4-5"
      },
      "models": {
        "anthropic/claude-sonnet-4-5": {
          "alias": "sonnet"
        }
      }
    }
  }
}
```

> 다른 필드는 건드리지 않는다. `model.primary`와 `models` 섹션만 수정.

### 6단계: auth-profiles 정리

필요시 `auth-profiles.json` 수정:
- 쿨다운 초기화: `cooldownUntil`, `failureCounts` 제거, `errorCount: 0`
- `lastGood` 업데이트: 선택된 프로필로 변경

### 7단계: 게이트웨이 재시작

```bash
openclaw gateway restart
```

### 8단계: 검증

재시작 후 상태 확인:
```bash
openclaw gateway health
```

사용자에게 결과 보고:
```
✅ 모델 전환 완료
  모델: anthropic/claude-sonnet-4-5 (sonnet)
  인증: OAuth token (anthropic:default)
  게이트웨이: 정상
```

## 트러블슈팅

### 전환 후 401 에러 발생 시
1. auth-profiles.json에서 해당 프로필의 쿨다운 상태 초기화
2. 토큰 길이 재검증
3. type/field 매칭 재확인
4. gateway restart 재실행

### 토큰이 잘리는 경우
터미널에서 토큰을 복사할 때 줄바꿈이 포함될 수 있음.
사용자에게 토큰을 한 줄로 이어붙여 입력하도록 안내.

### 모델명 오류
`openclaw configure --section model`의 인터랙티브 picker에서
UI 텍스트(`│  ◻ amazon-bedrock/...`)가 모델명으로 들어가는 버그 있음.
항상 이 스킬을 통해 직접 편집할 것.

---

*생성일: 2026-02-04*
*파일 위치: ~/.openclaw/workspace/skills/model-switch/SKILL.md*
