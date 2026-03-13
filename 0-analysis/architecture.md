# Discrawl 아키텍처 분석

## 모듈 구조

```
cmd/discrawl/main.go          ← 엔트리포인트 (cli.Run 호출)

internal/
├── cli/                       ← CLI 라우팅 + 출력 포맷팅
│   ├── cli.go                 ← runtime 구조체, dispatch, withServices
│   ├── admin_commands.go      ← init, sync, tail, doctor
│   ├── query_commands.go      ← search, sql, members, channels, status
│   ├── messages.go            ← messages 서브커맨드
│   ├── mentions.go            ← mentions 서브커맨드
│   ├── output.go              ← JSON/plain/human 출력
│   └── helpers.go             ← 유틸리티
│
├── config/                    ← TOML 설정 + 토큰 해석
│   └── config.go              ← Config 구조체, OpenClaw 토큰 연동
│
├── discord/                   ← Discord REST API + Gateway 래퍼
│   └── client.go              ← Client 구조체 (discordgo 기반)
│
├── syncer/                    ← 핵심 동기화 엔진
│   ├── syncer.go              ← Syncer 구조체, Sync() 오케스트레이터
│   ├── message_sync.go        ← 채널별 메시지 동기화 (forward/backfill)
│   ├── tail.go                ← Gateway 실시간 이벤트 핸들러
│   ├── channel_catalog.go     ← 채널/스레드 열거
│   ├── enrichment.go          ← 메시지 정규화 (embed, poll, attachment 평탄화)
│   └── records.go             ← Discord 객체 → DB 레코드 변환
│
└── store/                     ← SQLite 저장소
    ├── store.go               ← 스키마 마이그레이션, FTS5 인덱스
    ├── write.go               ← Upsert 로직
    ├── query.go               ← FTS 검색 + 멤버 쿼리
    ├── mentions.go            ← 멘션 추출/인덱싱
    └── messages.go            ← 메시지 CRUD
```

## 핵심 설계 패턴

### 1. Client 인터페이스 추상화

```go
type Client interface {
    Self(ctx) (*User, error)
    Guilds(ctx) ([]*UserGuild, error)
    Guild(ctx, string) (*Guild, error)
    GuildChannels(ctx, string) ([]*Channel, error)
    GuildMembers(ctx, string) ([]*Member, error)
    ChannelMessages(ctx, string, int, string, string) ([]*Message, error)
    Tail(ctx, EventHandler) error
}
```

**Syncer는 이 인터페이스에만 의존.** Discord 구현체를 Slack 구현체로 교체하면 syncer 로직은 그대로 재사용 가능... 이론상. 실제로는 Discord 타입(`*discordgo.Message` 등)이 syncer 내부에 깊이 침투해 있어서 완전 교체는 불가. **레코드 변환 레이어(records.go)를 Slack용으로 다시 작성해야 함.**

### 2. 양방향 동기화 (Sync + Tail)

```
[Discord REST API]          [Discord Gateway WebSocket]
       │                              │
  sync (backfill)               tail (realtime)
       │                              │
       ▼                              ▼
   ┌─────────────────────────────────────┐
   │           SQLite (WAL mode)         │
   │  messages, channels, members, FTS5  │
   └─────────────────────────────────────┘
```

- **Sync**: REST API로 과거 데이터 백필. forward(최신→) + backfill(←과거) 양방향 페이지네이션
- **Tail**: Gateway WebSocket으로 실시간 이벤트 수신. 주기적 repair sync(기본 6시간)로 누락 보정

### 3. 채널별 Sync State 관리

```
sync_state 테이블:
  scope = "channel:<id>:latest"           → 가장 최근 메시지 ID
  scope = "channel:<id>:backfill_cursor"  → 백필 진행 위치
  scope = "channel:<id>:history_complete" → 백필 완료 여부
  scope = "sync:last_success"             → 마지막 성공 시각
  scope = "tail:last_event"               → 마지막 tail 이벤트
```

**Resumable sync**: 중단 후 재개 시 cursor 위치부터 이어서 진행. Discord의 Snowflake ID가 시간순이므로 `maxSnowflake()`로 비교.

### 4. 메시지 정규화 (Enrichment)

`enrichment.go`에서 Discord 메시지의 다양한 형태를 단일 `normalized_content`로 평탄화:

- embed → 제목+설명+필드 텍스트 추출
- poll → 질문+선택지 텍스트
- attachment → 파일명 + (텍스트 파일이면 내용까지 추출)
- reply → 원본 메시지 참조

### 5. FTS5 전문 검색

```sql
CREATE VIRTUAL TABLE message_fts USING fts5(
    message_id UNINDEXED,
    guild_id UNINDEXED,
    channel_id UNINDEXED,
    author_id UNINDEXED,
    author_name,    -- 검색 대상
    channel_name,   -- 검색 대상
    content         -- 검색 대상
);
```

- `UNINDEXED` 컬럼은 저장만, 검색에는 사용 안 함 (조인 키용)
- rowID는 Discord Snowflake ID를 그대로 사용 (정수 파싱 가능 시) 또는 FNV 해시
- FTS 재빌드 시 버전 관리 (`schema:message_fts_rowid_version`)

### 6. SQLite 설정

```
journal_mode=WAL          ← 동시 읽기 허용
synchronous=NORMAL        ← 성능/안전 밸런스
mmap_size=268435456       ← 256MB 메모리 매핑
busy_timeout=5000         ← 5초 대기
MaxOpenConns=1            ← single-writer 보장
```

파일 권한 0o600 (소유자만 읽기/쓰기).

### 7. 동시 채널 동기화

`syncMessageChannelsConcurrent()`에서 worker pool 패턴:

```
                  ┌── worker 1 → syncChannelMessages(ch1) ──┐
jobs channel ────►├── worker 2 → syncChannelMessages(ch2) ──├──► results channel
                  └── worker N → syncChannelMessages(chN) ──┘
```

기본 concurrency: `min(32, max(8, GOMAXPROCS*2))`. 첫 에러 발생 시 `cancel()`로 전체 중단.

## DB 스키마 요약

| 테이블 | 용도 | 키 |
|--------|------|-----|
| `guilds` | 길드 메타데이터 | id (PK) |
| `channels` | 채널+스레드 통합 | id (PK) |
| `members` | 현재 스냅샷 (append-only 아님) | (guild_id, user_id) |
| `messages` | 메시지 본문 + normalized_content | id (PK) |
| `message_events` | 생명주기 이벤트 (create/update/delete) | event_id (auto) |
| `message_attachments` | 첨부 파일 메타 + 텍스트 추출 | attachment_id (PK) |
| `mention_events` | 멘션 인덱스 | event_id (auto) |
| `sync_state` | 동기화 체크포인트 | scope (PK) |
| `embedding_jobs` | 임베딩 대기열 | message_id (PK) |
| `message_fts` | 전문 검색 (FTS5 가상 테이블) | rowid |
| `member_fts` | 멤버 검색 (FTS5) | rowid |

## 설정 구조 (TOML)

```toml
version = 1
default_guild_id = "..."
db_path = "~/.discrawl/discrawl.db"

[discord]
token_source = "openclaw"    # openclaw | env
openclaw_config = "~/.openclaw/openclaw.json"
token_env = "DISCORD_BOT_TOKEN"

[sync]
concurrency = 16
repair_every = "6h"
full_history = true
attachment_text = true

[search]
default_mode = "fts"

[search.embeddings]
enabled = false
provider = "openai"
model = "text-embedding-3-small"
```

토큰 우선순위: OpenClaw config → 환경변수 → 에러.
