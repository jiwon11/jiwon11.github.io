# Post Images

블로그 포스트별 이미지를 저장하는 디렉토리입니다.

## 구조

```
posts/
└── YYYY-MM-DD-post-slug/
    ├── thumbnail.jpg      # 썸네일 이미지
    ├── diagram-*.svg      # SVG 다이어그램
    └── screenshot-*.png   # 스크린샷
```

## 이미지 유형

| 유형 | 형식 | 용도 |
|------|------|------|
| 썸네일 | jpg/png | 포스트 대표 이미지 |
| 다이어그램 | svg | 아키텍처, 플로우차트 |
| 스크린샷 | png | 에러 화면, UI 캡처 |

## 사용법

포스트에서 이미지 참조:

```markdown
![설명](/assets/images/posts/2026-01-29-example/thumbnail.jpg)
```
