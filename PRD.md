🧠 Product Requirements Document (PRD)

Project: AutoFix Pipeline

⸻

1. 개요 (Overview)

AutoFix Pipeline은
개발자가 관리하는 코드 리포지토리에서 이슈가 등록되면 자동으로 코드 수정 → PR 생성 → 알림 발송까지 수행하는 지능형 코드 자동 수정 시스템이다.

초기 버전은 GitHub 이슈를 트리거로 하며,
코드 분석 및 수정 제안을 수행한 뒤,
자동 브랜치 생성 → 커밋 → Pull Request 생성 → Slack 알림 전송을 완결적으로 처리한다.

이후 단계에서 이슈 소스(Jira), VCS(GitLab/Bitbucket), 알림 채널(Teams/Email) 로 확장이 가능하도록 설계된다.

⸻

2. 문제 정의 (Problem Statement)

구분 현행 문제 AutoFix가 해결하는 방식
반복적 코드 수리 단순한 Lint/Type/Error 수정이 반복되어 개발 생산성 저하 린트/타입 기반 자동 수정으로 단순 작업 제거
이슈 → PR 전환 지연 이슈 생성 후 담당자가 직접 브랜치/PR을 수동 처리 이슈를 트리거로 자동 브랜치·PR 생성
커뮤니케이션 지연 수정 완료 후 Slack/이슈 코멘트로 결과 공유 누락 Slack 알림 + 이슈 코멘트로 자동 리포팅
다양한 툴 연동 복잡 Jira, GitHub, Slack 등 API가 상이 어댑터 패턴으로 교체 가능한 모듈 구조 확보

⸻

3. 목표 (Objectives)
   1. 이슈 → 코드 수정 → PR 전 과정을 5분 내 자동화
   2. PR 자동 생성 성공률 95% 이상 (충돌 없는 브랜치)
   3. 수정 코드 빌드 성공률 90% 이상 (린트/테스트 통과)
   4. Slack 실시간 알림 100% 발송 (오류시 재시도 포함)

⸻

4. 주요 기능 (Key Features)

구분 기능명 설명
이슈 감지 Webhook Listener GitHub 이슈(또는 Jira) 이벤트 수신
작업 분류 Issue Classifier 라벨/키워드 기반으로 작업 타입 결정 (lint, type, deps 등)
코드 수정 AutoFix Engine 정적분석 기반 자동 수정 + 제한적 LLM diff 생성
브랜치/PR VCS Adapter 브랜치 생성, 커밋, PR 생성 (현재 GitHub)
알림 발송 Notifier Adapter Slack 메시지 전송 (추후 Teams/Email 추가)
대시보드 Web UI 진행 중 Job, PR, 로그 확인 및 리트라이
오케스트레이션 FastAPI + Worker 이벤트 관리, 작업 스케줄링, 상태 관리

⸻

5. 시스템 구조 (Architecture)

┌───────────────────────────┐
│ GitHub (Issue/PR) │
└──────────────┬────────────┘
│ Webhook
▼
[autofix-api (FastAPI)]
│
┌───────┴────────┐
▼ ▼
[autofix-worker] [autofix-web]
(코드 수정/PR) (대시보드)
│
▼
Slack 알림 (Notifier)

⸻

6. 기술 스택 (Tech Stack)

구분 기술 역할
Frontend Next.js 15 (App Router) 대시보드 UI
Backend API FastAPI Webhook 수신, Job 관리, 로그 API
Worker Python (async) 실제 코드 수정/테스트/커밋 처리
DB/Queue Redis / PostgreSQL Job 상태 관리
Contracts SDK OpenAPI + codegen TS/Python SDK 생성
Adapters Python packages Issues/VCS/Notify 모듈 (교체형)
Notifier Slack Web API 메시지/블록 전송
CI/CD GitHub Actions 릴리즈/테스트/버전관리 자동화

⸻

7. 데이터 흐름 (Flow)
   1. GitHub 이슈 생성/수정 → Webhook 전송
   2. API
      • payload 파싱
      • 관련 프로젝트/레포 확인
      • 작업 큐 등록 (Redis)
   3. Worker
      • 레포 클론 → 브랜치 생성
      • autofix-engine 실행 (자동수정)
      • 커밋 & 푸시 → PR 생성
      • Slack에 메시지 전송
   4. Web
      • Job 상태 실시간 표시
      • PR 링크 및 테스트 결과 확인

⸻

8. 인터페이스 정의

8.1 IssueProvider (어댑터)

class IssueProvider(Protocol):
def parse_event(self, payload: dict) -> IssueEvent
def fetch(self, key: str) -> Issue
def comment(self, key: str, body: str)

8.2 VCSHost (어댑터)

class VCSHost(Protocol):
def create_branch(self, repo: str, base: str, branch: str)
def commit_and_push(self, path: str, branch: str, message: str)
def open_pr(self, repo: str, base: str, head: str, title: str, body: str)

8.3 Notifier (어댑터)

class Notifier(Protocol):
def send(self, channel: str, text: str, blocks: list|None=None)

⸻

9. 비기능 요구사항 (Non-Functional Requirements)

항목 요구사항
성능 PR 생성까지 5분 이내
안정성 실패 시 재시도 3회, 중복 방지(Idempotent)
확장성 어댑터 기반, API 확장 시 기존 서비스 영향 없음
보안 API 키 암호화 저장, GitHub App 권한 최소화
가시성 Job 로그/상태를 API 및 Web UI에서 조회 가능

⸻

10. 성공 지표 (KPI)

지표 목표
PR 생성 성공률 ≥ 95%
빌드/테스트 통과율 ≥ 90%
Slack 알림 전송 성공률 100%
이슈 → PR 처리 시간 ≤ 10분
자동수정 정확도(적용 승인율) ≥ 80%

⸻

11. 단계별 개발 계획

단계 목표
Phase 1 GitHub 이슈 감지 + 브랜치/PR 자동화 + Slack 알림
Phase 2 AutoFix Engine(린트/타입 수정) + Web 대시보드
Phase 3 Jira 어댑터 추가 + Teams/Email 알림
Phase 4 CI 게이트 통합 + e2e 자동 테스트

⸻

12. 리스크 관리

리스크 완화 방안
GitHub API rate limit 캐시 및 큐 처리, exponential backoff
코드 충돌 브랜치 네이밍 규칙 + 최신 base pull 자동화
Slack API 오류 메시지 큐 + 재시도 + DLQ(Dead Letter Queue)
LLM 부정확성 제한된 diff 범위, 정적 분석 우선 적용

⸻

13. 향후 확장
    • Jira 어댑터: REST + Webhook 통합
    • GitLab 어댑터: self-hosted 환경 지원
    • Teams/Email 알림: 조직 단위 알림 연동
    • LLM 개선: LangChain 기반 컨텍스트 유지형 수정
    • 멀티레포 오케스트레이션: 동일 이슈 → 여러 레포 자동 수정

⸻

14. 성공 비전 (Vision)

AutoFix Pipeline은 단순한 “자동 코드 수정기”가 아니라
“이슈로부터 학습해 스스로 개선되는 개발 자동화 에이전트”다.

장기적으로는 코드리뷰·PR품질평가·이슈원인추적까지 통합한
AI-DevOps Copilot 플랫폼으로 확장될 수 있다.
