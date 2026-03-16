# Claude Ready Standards

Looking for the English version? [Read this document in English](README.md).

> Claude Code에 최적화된 사전 구축형 기술 표준.
> 기술 스택 결정에 시간을 낭비하지 마세요 — PRD가 준비되면 바로 구현을 시작하세요.

---

## Claude Ready Standards를 만든 이유

PRD와 기술 설계가 완료된 후에도, 구체적인 기술 선택과 기반 아키텍처 구성, 툴체인 설정에 상당한 시간과 에너지가 소모됩니다.

**Claude Ready Standards는 그 결정 피로를 제거합니다.** 이 저장소는 Claude Code와 함께 즉시 사용할 수 있는 기술 표준, 보일러플레이트 구성, 아키텍처 패턴을 제공합니다.

- **문제**: PRD가 있어도 팀은 기술 스펙, 폴더 구조, 기본 설정을 결정하는 데 며칠을 소비합니다.
- **해결**: 설계에서 구현까지 Claude Code로 즉시 전환할 수 있는 포괄적이고 바로 적용 가능한 표준 프레임워크.
- **범위**: 모노레포 구성, 백엔드/프론트엔드/클라이언트 설정, 테스팅 전략, 보안, CI/CD, Git 워크플로우.

---

## Claude Code 전용

> **주의**: 이 저장소의 모든 자동화 워크플로우는 **Claude Code 전용**으로 설계되었습니다.

Claude Ready Standards에 정의된 규칙, 에이전트 구성, 훅 설정, CLAUDE.md 패턴은 Claude Code의 기능에 완전히 최적화되어 있습니다.

다른 AI 코딩 도구를 사용하는 경우:

- CLAUDE.md의 계층적 구조가 무시됩니다.
- Rules, Hooks, Agents 구성이 동작하지 않습니다.
- 커밋 메시지 포맷, 코드 리뷰, TDD 워크플로우의 자동 적용이 보장되지 않습니다.
- **표준의 일관성과 구조가 훼손될 수 있습니다.**

---

## 주요 특징

### 19개 문서로 전체 라이프사이클 커버

```text
모노레포 셋업 → 공유 패키지 → 백엔드(NestJS/FastAPI/Kotlin) →
DB(PostgreSQL/MongoDB) → 큐 워커 → 인증/보안 → 프론트엔드(Next.js) →
모바일(Flutter) → Docker 인프라 → 테스팅 → CI/CD → 보안 체크리스트 →
PWA → WebSocket/실시간 → AI 에이전트 통합(CLAUDE.md)
```

### 3개 백엔드 언어 지원

하나의 표준 프레임워크 안에서 세 가지 백엔드를 통합합니다:

| 언어 | 프레임워크 |
|------|-----------|
| TypeScript | NestJS |
| Python | FastAPI |
| Kotlin | Spring Boot |

### AI 에이전트를 위한 문서 설계 원칙

| 원칙 | 의미 |
|------|------|
| **Blockquote Summary** | 문서 상단 1-2줄 요약으로 AI가 즉시 맥락 파악 |
| **Copy-Paste Ready** | 모든 코드 블록이 완전한 형태로 즉시 적용 가능 |
| **Verification Section** | 문서 하단 검증 명령어로 AI가 셀프 체크 |
| **No Business Logic** | 비즈니스 로직 배제로 어떤 도메인에도 범용 적용 |
| **TECH-STACK.md as SSOT** | 버전 정보 단일 소스로 불일치 방지 |

---

## 표준 카탈로그

### 활성 표준

| 카테고리 | 디렉토리 | 설명 |
|----------|---------|------|
| Boilerplate Guide | [`boilerplate-guide/`](./boilerplate-guide/) | pnpm 모노레포 + NestJS + FastAPI + Kotlin + Next.js + Flutter 기반 풀스택 프로젝트 부트스트래핑 가이드 (19개 문서) |

### 향후 카테고리 (예정)

| 카테고리 | 설명 |
|----------|------|
| `api-design/` | REST/GraphQL API 설계 표준 |
| `deployment/` | 배포 전략 및 인프라 구성 |
| `code-style/` | 언어별 코드 스타일 가이드 |
| `incident-response/` | 인시던트 대응 절차 |

---

## 주요 참조 문서

| 문서 | 설명 |
|------|------|
| [`TECH-STACK.md`](./TECH-STACK.md) | 기술 스택 버전 기준 (Single Source of Truth) |
| [`CLAUDE.md`](./CLAUDE.md) | 이 저장소에서 Claude Code 사용 시 적용되는 규칙 |
| [`OPEN-SOURCE-STARTER-GUIDE.md`](./OPEN-SOURCE-STARTER-GUIDE.md) | Claude Code로 새 오픈소스 프로젝트를 부트스트랩하는 가이드 |
| [`SECURITY.md`](./SECURITY.md) | 보안 정책 및 취약점 보고 |
| [`boilerplate-guide/00-index.md`](./boilerplate-guide/00-index.md) | Boilerplate Guide 인덱스 |

---

## 기여

기여를 환영합니다! PR을 제출하기 전에 [기여 가이드](CONTRIBUTING.md)를 읽어주세요.

이 프로젝트는 [Contributor Covenant 행동 강령](CODE_OF_CONDUCT.md)을 따릅니다. 보안 관련 사항은 [보안 정책](SECURITY.md)을 참고하세요.

---

## 라이선스

이 저작물은 [Creative Commons Attribution 4.0 International (CC BY 4.0)](LICENSE) 라이선스 하에 배포됩니다.

적절한 저작자 표시를 하는 한, 상업적 목적을 포함하여 자유롭게 공유하고 수정할 수 있습니다.

---

## 저자

[kanjseok](https://github.com/kanjseok)이 제작 및 유지보수합니다.
