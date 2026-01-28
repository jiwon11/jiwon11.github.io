# Claude Code Conversations Backup

이 디렉토리는 Claude Code 대화 이력 백업을 저장합니다.

## 파일 구조

```
claude-conversations/
├── latest.json              # 최근 백업 메타데이터
├── conversation-YYYY-MM-DD.json   # JSON 형식 대화
└── conversation-YYYY-MM-DD.md     # Markdown 형식 대화
```

## 자동 백업

로컬 cron 스크립트가 매일 대화를 이 디렉토리에 백업합니다.

## GitHub Actions 연동

새 파일이 추가되면 `devlog-automation.yml` 워크플로우가 트리거되어:
1. 새 대화 감지
2. 이슈 자동 생성
3. 요약 파일 생성
