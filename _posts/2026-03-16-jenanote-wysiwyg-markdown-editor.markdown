---
layout: post
ref: jenanote-wysiwyg-markdown-editor
title: "JenaNote — NSDocument로 만드는 WYSIWYG 마크다운 에디터"
date: 2026-03-16 10:00:00
categories: macos
author: jenalab
image: /assets/article_images/jenanote/wysiwyg-editor.svg
image2: /assets/article_images/jenanote/wysiwyg-editor.svg
---

## 마크다운 기호가 보이지 않는 에디터

`**굵게**`를 입력하면 `**`가 사라지고 **굵게**만 남는다. `# 제목`을 입력하면 `#`이 사라지고 큰 글씨만 보인다. 저장하면 `.md` 파일이다. 다른 마크다운 뷰어에서 열어도 서식이 일치한다.

JenaNote는 이 동작을 수행하는 macOS 네이티브 에디터다.

## 문제: 기존 도구의 틈

마크다운 에디터는 두 부류로 나뉜다.

1. **기호 노출형** (Obsidian, VS Code). 마크다운 문법을 알아야 한다. `#`, `**`, `-` 기호가 화면에 그대로 보인다.
2. **서식 에디터** (TextEdit, Notes). WYSIWYG이지만 `.md`로 저장하지 못한다.

Typora가 이 틈을 메웠지만, 유료 전환 후 대안이 필요해졌다. JenaNote는 그 자리를 MIT 오픈소스로 채운다.

| 기준 | JenaNote | Typora | Obsidian | TextEdit |
|---|---|---|---|---|
| WYSIWYG | O | O | X | O |
| .md 저장 | O | O | O | X |
| 기호 숨김 | O | 부분 | X | 해당없음 |
| macOS 네이티브 | O | O | X | O |
| 오픈소스 | MIT | 유료 | 유료 | 시스템 앱 |

## 설계의 출발점: NSDocument

macOS에는 문서 기반 앱을 위한 표준 패턴이 있다. NSDocument다. 이 클래스를 상속하면 다음이 공짜로 따라온다.

- 저장 / 다른 이름으로 저장 대화상자
- 창 제목의 변경 표시 점(●)
- 닫기 전 "저장하시겠습니까?" 확인 시트
- 앱 종료 후 재시작 시 문서 복원
- Undo / Redo 스택

직접 구현하면 이 목록만으로 수백 줄이다. NSDocument를 쓰면 0줄이다.

**결정: NSDocument를 상속하고, 커스텀 파일 관리 코드를 작성하지 않는다.**

## 3레이어 아키텍처

```
┌─────────────────────────────────────────┐
│  UI Layer                               │
│  텍스트 뷰 · 서식 툴바 · 뷰컨트롤러      │
└────────────────┬────────────────────────┘
                 │ NSAttributedString 읽기/쓰기
┌────────────────▼────────────────────────┐
│  Document Layer                         │
│  마크다운 문서 (NSDocument 서브클래스)     │
└────────────────┬────────────────────────┘
                 │ 직렬화 / 역직렬화
┌────────────────▼────────────────────────┐
│  Infrastructure Layer                   │
│  마크다운 시리얼라이저                    │
│  (NSAttributedString ↔ CommonMark .md)  │
└─────────────────────────────────────────┘
```

의존성은 위에서 아래로만 흐른다. 세 가지 규칙을 정했다.

1. UI는 시리얼라이저를 직접 호출하지 않는다
2. Document는 AppKit UI 컴포넌트에 의존하지 않는다
3. 시리얼라이저는 순수 함수다. 상태가 없다

### 핵심 결정: 내부 표현으로 NSAttributedString

마크다운 에디터를 만들 때 내부 표현의 선택이 아키텍처 전체를 결정한다.

| 선택지 | 장점 | 단점 |
|---|---|---|
| 마크다운 AST | CommonMark 완전 지원 | NSTextView와 별도 동기화 필요 |
| NSAttributedString | NSTextView 직접 통합, Undo 자동 | 복잡한 중첩 구조에 한계 |

NSAttributedString을 선택했다. NSTextView가 이 형식을 네이티브로 처리한다. 볼드, 이탤릭, 제목, 목록을 속성(attribute)으로 저장하면 화면에 즉시 반영된다. Undo/Redo도 NSUndoManager가 자동 처리한다.

트레이드오프는 있다. 마크다운의 복잡한 중첩 구조(리스트 안의 코드 블록 안의 볼드)를 완벽하게 표현하기 어렵다. 메모 앱의 범위에서는 허용 가능한 제약이다.

## 데이터 흐름

### 파일 열기

```
Finder에서 .md 더블클릭
  → NSDocumentController가 Document 인스턴스 생성
  → 시리얼라이저: 마크다운 텍스트 → NSAttributedString
  → 뷰컨트롤러가 텍스트 뷰에 로드
  → 기호 없이 서식 적용된 상태로 표시
```

### 서식 적용

```
Cmd+B 또는 툴바 B 클릭
  → 뷰컨트롤러가 서식 명령 호출
  → 텍스트 스토리지에 속성 변경 (Undo 등록)
  → 화면 즉시 반영. 기호는 보이지 않는다.
```

### 저장

```
Cmd+S
  → NSDocument가 저장 처리 (AppKit 자동)
  → 시리얼라이저: NSAttributedString → CommonMark 텍스트
  → .md 파일로 디스크 기록
  → 창 제목의 ● 표시 제거
```

## 시리얼라이저: 양방향 변환

시리얼라이저는 두 가지 일을 한다.

**파싱**: `.md` 파일의 마크다운 텍스트를 받아 NSAttributedString으로 변환한다. `**텍스트**`는 볼드 속성이 적용된 "텍스트"가 된다.

**직렬화**: NSAttributedString의 속성을 읽어 CommonMark 텍스트로 변환한다. 볼드 속성이 있으면 `**`로 감싼다.

이 모듈은 순수 함수다. 입력을 받아 출력을 반환한다. 상태를 저장하지 않는다. Foundation만 import한다. 테스트가 쉬운 이유다.

## macOS 표준을 따르는 이유

### 메뉴와 단축키

macOS의 Responder Chain을 그대로 사용한다. 별도 메뉴 관리 코드가 필요 없다.

- 파일 메뉴(저장, 열기) → NSDocument가 자동 처리
- 편집 메뉴(Undo, 복사, 붙여넣기) → NSTextView가 자동 처리
- 서식 메뉴(Bold, Italic) → 커스텀 액션만 연결

커스텀 코드가 필요한 곳은 서식 적용뿐이다. 나머지는 AppKit이 처리한다.

### 미저장 경고

변경 사항이 있는 상태에서 창을 닫으면 확인 시트가 뜬다. "저장", "저장 안 함", "취소". 이 동작도 NSDocument가 자동 제공한다. 직접 구현하면 닫기, 종료, 새 문서 열기 등 모든 경로에서 경고를 호출해야 한다. 빠뜨리면 데이터 손실이다.

## 빌드: Xcode 없이

jenaMemory와 같은 방식이다. Makefile + `swiftc`.

```
make run      # 빌드 + 실행
make build    # .app 번들 생성
make install  # ~/Applications에 설치
make dmg      # 배포용 디스크 이미지
make pkg      # 패키지 인스톨러
```

Swift 파일을 하나의 바이너리로 컴파일한다. Xcode 프로젝트 파일이 없으므로 Git diff가 깨끗하다. 설정 변경이 Makefile 한 줄에 담긴다.

## 정리

WYSIWYG 마크다운 에디터를 만드는 데 필요한 것은 세 가지다.

1. **NSDocument** — 파일 관리를 macOS에 맡긴다
2. **NSAttributedString** — 내부 표현과 화면 표시를 일치시킨다
3. **양방향 시리얼라이저** — `.md` 파일과의 변환을 담당한다

나머지는 AppKit이 처리한다. 플랫폼이 제공하는 것을 다시 만들지 않는다. 이것이 네이티브 앱의 설계 원칙이다.

---

**JenaNote**는 MIT 오픈소스다.

[GitHub](https://github.com/jenalab-com/jena-note) · [DMG 다운로드](https://github.com/jenalab-com/jena-note/releases/latest/download/JenaNote-1.0.0.dmg)
