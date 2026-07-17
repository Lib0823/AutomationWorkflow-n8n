# Automate_News_Summary

n8n을 활용한 뉴스 요약 알림 및 워크플로우 자동 백업 시스템

## 워크플로우 구성

### 1. NewsNotificationScheduler

RSS 피드를 수집해 Gemini로 한국어 요약을 생성하고, 카카오톡 "나에게 보내기"로 전송하는 워크플로우. 경제 뉴스와 개발 뉴스 두 트랙이 독립적으로 동작한다.

| 구분 | 발송 시각 (KST) | 뉴스 소스 | 선정 기준 |
|------|-----------------|-----------|-----------|
| 경제 뉴스 | 08:00 | [CNBC RSS](https://search.cnbc.com/rs/search/combinedcms/view.xml?partnerId=wrss01&id=100003114) | 미국 증시·글로벌 경제 영향력 상위 3건 |
| 개발 뉴스 | 20:00 | [Hacker News Frontpage](https://hnrss.org/frontpage) | 기술 스택·오픈소스·아키텍처 관점 상위 3건 |

**처리 흐름**: Schedule Trigger → RSS Feed Read → Limit(10건) → Aggregate(title/contentSnippet/link 병합) → Google Gemini(`gemini-2.5-flash`)로 한국어 JSON 요약 생성 → HTTP Request(Kakao Talk Memo API)로 카카오톡 발송

각 Gemini 노드는 원문을 분석해 아래 형식의 JSON 배열만 출력하도록 프롬프트로 강제한다.

```json
[
  { "title": "[📍 한국어 제목]", "description": "150~300자 요약", "link": "기사 URL" }
]
```

HTTP Request 노드는 이 JSON을 파싱해 카카오톡 텍스트 메모 형식으로 재조합한 뒤 전송한다.

### 2. Git-CICD

n8n 인스턴스에 저장된 모든 워크플로우를 매일 이 GitHub 저장소에 백업하는 워크플로우.

**처리 흐름**: Schedule Trigger(매일 04:00 KST) → Get many workflows(n8n API) → Edit a file(GitHub API)로 `workflows/{워크플로우명}.json`에 커밋 (`Daily backup: yyyy-MM-dd`)

저장소의 `Daily backup` 커밋 이력은 이 워크플로우가 매일 자동 생성한 것이다.

## 필요 자격 증명 (n8n Credentials)

| 자격 증명 | 사용처 |
|-----------|--------|
| Google Gemini (PaLM) API | 뉴스 요약 생성 |
| Kakao OAuth2 (카카오톡 나에게 보내기) | 뉴스 알림 발송 |
| n8n API | 워크플로우 목록 조회 (Git-CICD) |
| GitHub API | 워크플로우 JSON 백업 커밋 (Git-CICD) |

## 디렉터리 구조

```
workflows/
├── NewsNotificationScheduler.json  # 경제/개발 뉴스 요약 및 카카오톡 발송
└── Git-CICD.json                   # 워크플로우 일일 자동 백업
```
