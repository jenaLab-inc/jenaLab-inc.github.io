---
layout: post
ref: jenamemory-macos-menubar-app
title: "jenaMemory — Xcode 없이 만드는 macOS 메뉴바 앱"
date: 2026-03-14 10:00:00
categories: macos
author: jenalab
image: /assets/article_images/jenamemory/menubar-app.svg
image2: /assets/article_images/jenamemory/menubar-app.svg
---

## 메뉴바에 메모리 사용량을 띄운다

macOS 메뉴바 오른쪽에 숫자 하나가 뜬다. 현재 여유 메모리 퍼센트다. 2초마다 갱신된다. 클릭하면 상세 통계가 나온다. 한 번 더 클릭하면 캐시를 비운다.

jenaMemory는 이 동작을 수행하는 경량 유틸리티다. Swift 파일 8개, Xcode 프로젝트 없이 `swiftc`로 빌드한다.

## 왜 직접 만들었나

메모리 모니터 앱은 App Store에 넘친다. 대부분 두 가지 문제가 있다.

1. **무겁다**. 메모리를 감시하는 앱이 메모리를 잡아먹는다.
2. **팝업이 많다**. 구독 유도, 프리미엄 잠금, 광고.

필요한 건 숫자 하나와 버튼 하나다. 그래서 직접 만들었다.

## 설계 결정

### AppKit, SwiftUI가 아닌 이유

SwiftUI는 macOS 13부터 안정적이다. 하지만 메뉴바 앱에서는 AppKit이 더 적합하다.

| 기준 | SwiftUI | AppKit |
|---|---|---|
| NSStatusItem 제어 | 래핑 필요 | 네이티브 |
| 메뉴 동적 갱신 | 번거로움 | 직관적 |
| 번들 크기 | 더 큼 | 최소 |
| macOS 12 이하 지원 | 불안정 | 안정 |

메뉴바 앱은 UI가 단순하다. SwiftUI의 선언적 장점이 발휘되지 않는다. AppKit으로 직접 제어하는 편이 가볍고 예측 가능하다.

### Xcode 없이 빌드하는 이유

```bash
swiftc -framework AppKit -O Sources/*.swift -o jenaMemory
```

한 줄이면 빌드된다. Xcode 프로젝트 파일(.xcodeproj)은 Git diff가 읽기 어렵다. 설정 변경이 XML 안에 묻힌다. Makefile로 빌드하면 모든 설정이 텍스트로 추적된다.

```
make run      # 빌드 + 즉시 실행
make build    # .app 번들 생성
make install  # ~/Applications에 설치
make pkg      # .pkg 인스톨러 생성
make dmg      # .dmg 배포 이미지 생성
```

## 아키텍처: 8개 파일, 역할 분리

```
┌──────────────────────────────────────────────┐
│  main.swift → AppDelegate                    │
│       │                                      │
│       ▼                                      │
│  MenuBarController (메인 컨트롤러)             │
│       │                                      │
│       ├── MemoryMonitor (시스템 메모리 조회)    │
│       ├── PrivilegedExecutor (권한 실행)       │
│       ├── SettingsWindowController (설정)      │
│       ├── AboutWindowController (정보)         │
│       └── Localization (9개 언어)              │
└──────────────────────────────────────────────┘
```

파일 수가 8개다. 의도적이다. 메뉴바 유틸리티에 레이어드 아키텍처나 MVVM은 과잉이다. 역할별로 파일을 나누되, 추상화 계층은 만들지 않았다.

### MenuBarController — 모든 것의 중심

이 파일이 앱의 중심이다. NSStatusItem을 생성하고, 2초 타이머로 메모리를 갱신하고, 메뉴를 구성한다.

메뉴바 아이콘은 RAM 칩 모양이다. 메모리 사용률에 따라 채워지는 비율이 달라진다. 라이트/다크 모드에 자동 대응한다. 16x18 픽셀에 이 모든 정보를 담는다.

### MemoryMonitor — Darwin Mach API 직접 호출

메모리 통계를 Foundation이 아닌 Darwin Mach API로 조회한다. `vm_statistics64`를 직접 호출한다. Foundation을 거치면 불필요한 오버헤드가 생긴다. 2초마다 호출되는 함수에서 오버헤드는 누적된다.

메모리 계산 모델은 macOS Activity Monitor와 일치시켰다.

| 항목 | 계산 |
|---|---|
| 사용 중 | Wired + Active + Compressed |
| 캐시 | Inactive (필요 시 회수 가능) |
| 여유 | Free + Inactive |
| 사용률 | 사용 중 ÷ 전체 × 100 |

Inactive 메모리를 "여유"에 포함한다. macOS는 Inactive를 캐시처럼 관리하며, 새 프로세스가 요청하면 즉시 회수한다.

### PrivilegedExecutor — 2단계 권한 상승

메모리 최적화는 시스템의 `purge` 명령을 실행한다. 관리자 권한이 필요하다.

```
첫 실행:
  AppleScript → 시스템 암호 입력창 → sudoers 규칙 설치 → purge 실행

이후 실행:
  sudoers 규칙 확인 → 암호 없이 purge 실행
```

첫 실행에서 한 번만 암호를 입력하면, 이후에는 묻지 않는다. sudoers 규칙을 설치하여 해당 명령만 무암호로 허용한다. 전체 sudo 권한을 열지 않는다.

임시 파일은 `defer`로 자동 삭제한다. 권한 관련 코드에서 임시 파일이 남으면 보안 위험이다.

## 자동 최적화

여유 메모리가 설정한 임계값 이하로 떨어지면 자동으로 최적화를 실행한다. 60초 쿨다운을 둔다. 이 쿨다운이 없으면 임계값 근처에서 최적화가 반복 실행된다.

```
2초마다 메모리 확인
  │
  ├── 여유 > 임계값 → 대기
  └── 여유 ≤ 임계값
       ├── 마지막 실행 후 60초 미경과 → 대기
       └── 60초 경과 → 자동 최적화 실행
```

## 다국어 지원: 9개 언어

한국어, 영어, 중국어, 일본어, 스페인어, 독일어, 프랑스어, 러시아어, 힌디어를 지원한다.

`.strings` 파일이나 `.lproj` 번들을 쓰지 않았다. Swift enum으로 모든 문자열을 관리한다. 메뉴바 앱에서 Xcode 로컬라이제이션 시스템은 과잉이다.

언어를 변경하면 메뉴, 설정 창, 정보 창이 즉시 갱신된다. 앱 재시작이 필요 없다. NotificationCenter로 변경 이벤트를 전파하고, 각 컨트롤러가 UI를 다시 그린다.

## Dock에 아이콘이 없다

`LSUIElement = true` 설정으로 Dock에서 숨긴다. 메뉴바 앱은 Dock에 있을 이유가 없다. 항상 메뉴바에 있고, 설정 창을 열 때만 포커스를 받는다. macOS의 시스템 유틸리티들이 따르는 패턴이다.

## 배포: DMG와 PKG

```
make dmg    # 디스크 이미지 (.dmg)
make pkg    # 패키지 인스톨러 (.pkg)
```

두 형식 모두 Makefile 한 줄로 생성한다. 코드 서명과 노터라이제이션은 별도 처리가 필요하지만, 빌드 자체는 자동화되어 있다.

## 정리

메뉴바 유틸리티에 필요한 것은 많지 않다. AppKit, Mach API, Makefile. 8개 파일. 외부 의존성 0개. Xcode 프로젝트 0개.

복잡한 도구가 필요하지 않을 때, 도구를 줄이는 것도 설계다.

---

**jenaMemory**는 MIT 오픈소스다.

[GitHub](https://github.com/jenalab-com/jenaMemory) · [DMG 다운로드](https://github.com/jenalab-com/jenaMemory/releases/latest/download/jenaMemory-1.0.0.dmg)
