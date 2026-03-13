# Discrawl 소스 분석

steipete/discrawl — Discord 서버 데이터를 로컬 SQLite에 아카이빙하고 전문 검색하는 Go CLI 도구.

## 분석 목적

Slack 버전 아카이버(slacrawl)를 만들기 위해 discrawl의 아키텍처를 심층 분석.

## 문서 목록

| 파일 | 내용 |
|---|---|
| [architecture.md](./architecture.md) | 전체 아키텍처, 모듈 구조, 데이터 흐름 |
| [slack-adaptation.md](./slack-adaptation.md) | Slack 버전 설계 — Discord↔Slack API 매핑, 변경 필요 지점 |

## 분석 대상 버전

- discrawl v0.x (2026년 3월 기준 최신, MIT 라이선스)
- 핵심 소스: `internal/` (syncer, store, cli, config, discord)
