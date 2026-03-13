# Slack 버전 설계 (slacrawl)

discrawl 아키텍처를 기반으로 Slack 워크스페이스 아카이버를 만드는 설계 문서.

## Discord ↔ Slack 개념 매핑

| Discord | Slack | 비고 |
|---------|-------|------|
| Guild | Workspace | 최상위 단위. Slack은 보통 1개 workspace |
| Channel | Channel | 거의 동일 |
| Thread (in channel) | Thread (in channel) | Slack은 `thread_ts`로 스레드 식별 |
| Forum channel | — | Slack에 없음 |
| Category | — | Slack은 채널 섹션(폴더)이 있지만 API로 접근 불가 |
| Member | Member | 유사. Slack은 profile 필드가 더 풍부 |
| Message ID (Snowflake) | `ts` (timestamp) | Slack은 `channel_id + ts`가 유니크 키 |
| Gateway WebSocket | Socket Mode / Events API | 실시간 이벤트 수신 방식 다름 |
| Bot Token | Bot Token (xoxb-) | 유사. Slack은 OAuth scope 기반 |
| User Token | User Token (xoxp-) | Slack은 사용자 토큰으로 더 많은 데이터 접근 가능 |

## 핵심 차이점 (설계에 영향)

### 1. 메시지 ID 체계

- **Discord**: Snowflake ID (정수, 시간순 정렬 가능, 전역 유니크)
- **Slack**: `ts` (타임스탬프 문자열, 예: `"1234567890.123456"`, 채널 내에서만 유니크)

**영향**:
- `maxSnowflake()` 비교 로직 → `maxTimestamp()` 문자열 비교로 변경
- FTS rowID를 Snowflake에서 직접 생성하던 로직 → ts를 정수로 변환하거나 autoincrement
- sync_state의 cursor가 `(channel_id, ts)` 쌍이 되어야 함

### 2. 페이지네이션

- **Discord**: `before=<message_id>`, `after=<message_id>`, limit 100
- **Slack**: `conversations.history` — `cursor` 기반 (opaque string), `oldest`/`latest` ts, limit 1000

**영향**:
- Slack은 커서 기반이라 backfill이 더 단순 (cursor 따라가면 됨)
- `oldest`/`latest` 파라미터로 시간 범위 지정 가능 → incremental sync가 더 쉬움
- limit가 1000까지 → 페이지 수 줄어듦 (Discord는 100)

### 3. 실시간 이벤트

- **Discord**: Gateway WebSocket (discordgo가 관리)
- **Slack**: 두 가지 방식
  - **Socket Mode**: WebSocket 기반, 서버 불필요, App-Level Token 필요
  - **Events API**: HTTP webhook, 서버 필요

**권장**: Socket Mode (서버 인프라 불필요, discrawl의 tail과 가장 유사)

### 4. Rate Limiting

- **Discord**: 글로벌 + 라우트별 rate limit, 429 + Retry-After 헤더
- **Slack**: Tier 1-4 rate limit, 429 + Retry-After

**영향**: 둘 다 429 핸들링 필요. Slack은 `conversations.history`가 Tier 3 (50+/min)이라 대규모 백필 시 주의.

### 5. 스레드

- **Discord**: 스레드는 별도 채널 (channel type으로 구분)
- **Slack**: 스레드는 부모 메시지의 `thread_ts`로 식별, `conversations.replies`로 별도 조회

**영향**: 채널 목록에 스레드가 포함되지 않음. 메시지에 `reply_count > 0`이면 별도로 스레드 내용을 가져와야 함. discrawl의 `channel_catalog.go` 대신 메시지 동기화 중에 스레드를 발견하고 추가 fetch하는 로직 필요.

### 6. 권한/Scope

- **Discord Bot**: `VIEW_CHANNEL`, `READ_MESSAGE_HISTORY` 정도
- **Slack Bot**: 필요 scope:
  - `channels:history` — public 채널 메시지 읽기
  - `channels:read` — 채널 목록
  - `groups:history` — private 채널 메시지 (선택)
  - `groups:read` — private 채널 목록 (선택)
  - `users:read` — 멤버 정보
  - `users.profile:read` — 프로필 상세
  - `team:read` — 워크스페이스 정보

## 모듈별 변경 계획

### `internal/slack/client.go` (신규, discord/client.go 대체)

```go
type Client interface {
    Self(ctx) (*slack.AuthTestResponse, error)
    Workspace(ctx) (*slack.TeamInfo, error)
    Channels(ctx) ([]slack.Channel, error)  // conversations.list
    Members(ctx) ([]slack.User, error)       // users.list
    ChannelHistory(ctx, channelID, cursor, oldest, latest string, limit int) ([]slack.Message, string, error)
    ThreadReplies(ctx, channelID, threadTS string) ([]slack.Message, error)
    Tail(ctx, EventHandler) error            // Socket Mode
}
```

Slack Go SDK: `github.com/slack-go/slack` + `github.com/slack-go/slack/socketmode`

### `internal/syncer/` (대부분 재사용 가능)

| 파일 | 변경 수준 | 설명 |
|------|----------|------|
| `syncer.go` | 중간 | Guild→Workspace, 타입 변경 |
| `message_sync.go` | 높음 | Snowflake→ts, 페이지네이션 방식 변경, 스레드 fetch 추가 |
| `tail.go` | 높음 | Gateway→Socket Mode 핸들러 |
| `channel_catalog.go` | 낮음 | conversations.list로 단순화 (스레드 별도) |
| `enrichment.go` | 중간 | Slack 메시지 포맷에 맞게 (blocks, attachments, files) |
| `records.go` | 높음 | Discord 타입 → Slack 타입 변환 전면 재작성 |

### `internal/store/` (거의 그대로)

스키마 변경:

```sql
-- guilds → workspaces
CREATE TABLE workspaces (
    id TEXT PRIMARY KEY,        -- team_id
    name TEXT NOT NULL,
    domain TEXT,                -- xxx.slack.com
    raw_json TEXT NOT NULL,
    updated_at TEXT NOT NULL
);

-- channels (유사, 약간 변경)
CREATE TABLE channels (
    id TEXT PRIMARY KEY,
    workspace_id TEXT NOT NULL,
    kind TEXT NOT NULL,          -- public_channel | private_channel | mpim | im
    name TEXT NOT NULL,
    topic TEXT,
    purpose TEXT,               -- Slack 고유
    is_archived INTEGER NOT NULL DEFAULT 0,
    is_private INTEGER NOT NULL DEFAULT 0,
    raw_json TEXT NOT NULL,
    updated_at TEXT NOT NULL
);

-- messages (핵심 차이: id = channel_id + ts)
CREATE TABLE messages (
    channel_id TEXT NOT NULL,
    ts TEXT NOT NULL,            -- Slack timestamp (메시지 ID)
    workspace_id TEXT NOT NULL,
    author_id TEXT,
    thread_ts TEXT,              -- 스레드 부모 ts (NULL이면 일반 메시지)
    reply_count INTEGER NOT NULL DEFAULT 0,
    created_at TEXT NOT NULL,
    edited_at TEXT,
    deleted_at TEXT,
    content TEXT NOT NULL,
    normalized_content TEXT NOT NULL,
    has_files INTEGER NOT NULL DEFAULT 0,
    raw_json TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    PRIMARY KEY (channel_id, ts)
);

-- FTS: channel_id + ts를 키로
CREATE VIRTUAL TABLE message_fts USING fts5(
    channel_id UNINDEXED,
    ts UNINDEXED,
    workspace_id UNINDEXED,
    author_id UNINDEXED,
    author_name,
    channel_name,
    content
);
```

### `internal/config/` (Slack 토큰 방식으로)

```toml
version = 1
db_path = "~/.slacrawl/slacrawl.db"

[slack]
token_source = "env"           # env | openclaw
bot_token_env = "SLACK_BOT_TOKEN"
app_token_env = "SLACK_APP_TOKEN"   # Socket Mode용

[sync]
concurrency = 4                # Slack rate limit 고려, Discord보다 낮게
repair_every = "6h"
full_history = true
include_private = false        # private 채널 포함 여부
include_threads = true         # 스레드 내용 포함 여부

[search]
default_mode = "fts"
```

### `internal/cli/` (명령어 동일, 출력 적응)

CLI 인터페이스는 거의 동일하게 유지:

```
slacrawl init          # workspace 발견, 설정 생성
slacrawl sync --full   # 전체 백필
slacrawl tail          # Socket Mode 실시간 수신
slacrawl search "키워드"
slacrawl sql "SELECT ..."
slacrawl members list
slacrawl channels list
slacrawl status
slacrawl doctor
```

## 스레드 동기화 전략

Discord와 가장 다른 부분. Slack 스레드는 별도 채널이 아니라 부모 메시지 하위.

```
syncChannelMessages(channel):
    for each page in conversations.history:
        for each message:
            upsert message
            if message.reply_count > 0:
                replies = conversations.replies(channel, message.thread_ts)
                for each reply:
                    upsert reply (thread_ts = parent.ts)
```

**최적화**: reply_count가 변하지 않았으면 스레드 재fetch 스킵.

## Rate Limit 전략

| API | Tier | 제한 | 대응 |
|-----|------|------|------|
| `conversations.list` | Tier 2 | 20/min | 한 번 호출이면 충분 |
| `conversations.history` | Tier 3 | 50/min | 채널 간 0.5~1초 간격 |
| `conversations.replies` | Tier 3 | 50/min | 스레드 많으면 병목 |
| `users.list` | Tier 2 | 20/min | 한 번 호출 |

**구현**: 429 응답 시 `Retry-After` 헤더 대기. `golang.org/x/time/rate` Limiter 사용.

## 구현 우선순위

1. **Phase 1**: slack client + 설정 + sync (채널 메시지만, 스레드 제외)
2. **Phase 2**: FTS 검색 + CLI 쿼리 명령어
3. **Phase 3**: 스레드 동기화
4. **Phase 4**: Socket Mode tail + repair sync
5. **Phase 5**: 임베딩 검색 (선택)

## 재사용 가능한 코드 (변경 없이 또는 최소 변경)

- `store/store.go` — SQLite 열기/닫기, WAL 설정, 마이그레이션 프레임워크
- `store/query.go` — FTS 검색 로직 (컬럼명만 조정)
- `cli/cli.go` — runtime 구조체, dispatch, withServices 패턴
- `cli/output.go` — JSON/plain/human 출력 포맷팅
- `config/config.go` — TOML 파싱, 경로 확장, 기본값 패턴
- `syncer/message_sync.go` — forward/backfill 페이지네이션 골격 (ID 비교 로직만 변경)

## 필요한 Slack App 설정

1. https://api.slack.com/apps 에서 앱 생성
2. **Bot Token Scopes**: `channels:history`, `channels:read`, `users:read`, `users.profile:read`, `team:read`
3. **Socket Mode** 활성화 → App-Level Token 발급 (`connections:write` scope)
4. **Event Subscriptions** (Socket Mode): `message.channels`, `message.groups` (선택)
5. 워크스페이스에 앱 설치 → Bot Token (`xoxb-...`) 획득
