---
layout: post
ref: jenaimage-native-image-viewer
title: "JenaImage — Finder와 Preview 사이를 없앤 macOS 이미지 뷰어"
date: 2026-03-18 10:00:00
categories: macos
author: jenalab
image: /assets/article_images/jenaimage/image-viewer.svg
image2: /assets/article_images/jenaimage/image-viewer.svg
---

## 이미지를 관리하려면 앱 두 개가 필요하다

폴더를 탐색한다. Finder. 이미지를 본다. Preview. 다른 폴더로 옮긴다. 다시 Finder. 포맷을 변환한다. 또 다른 앱.

이미지 작업은 항상 앱 사이를 오간다. JenaImage는 이 이동을 없앤다. 폴더 사이드바, 썸네일 그리드, 이미지 뷰어, 동영상 재생, 포맷 변환이 한 창에 있다.

## 설계 결정

### AppKit을 선택한 이유

이 앱은 세 가지 핵심 컴포넌트를 사용한다.

| 컴포넌트 | AppKit | SwiftUI |
|---|---|---|
| 폴더 트리 | NSOutlineView — 안정적 | OutlineGroup — 대규모 트리에서 불안정 |
| 이미지 그리드 | NSCollectionView — 성능 검증됨 | LazyVGrid — 스크롤 끊김 보고 |
| 줌/팬 뷰어 | NSScrollView magnification | ScrollView — 세밀한 제어 어려움 |

세 컴포넌트 모두 AppKit이 우위다. macOS 14 이상을 타겟으로 해도 SwiftUI의 이 영역은 성숙하지 않았다.

### Swift Concurrency, GCD가 아닌 이유

이미지 뷰어는 비동기 작업이 많다. 폴더 탐색, 썸네일 생성, 대용량 이미지 로딩, 포맷 변환. 모두 백그라운드에서 처리해야 한다.

GCD 대신 Swift Concurrency(`async/await`)를 택했다. 이유는 하나다. **작업 취소**. 폴더를 전환하면 이전 폴더의 썸네일 생성을 즉시 중단해야 한다. GCD에서는 취소 로직을 직접 구현해야 한다. Swift Concurrency는 `Task.isCancelled`로 해결된다.

## 아키텍처: 한 창 안의 세 영역

```
┌──────────────────────────────────────────────────────┐
│  MainWindowController (NSSplitViewController)        │
├──────────┬───────────────────────────────────────────┤
│ Sidebar  │  Browser (그리드) ←→ Viewer (상세)        │
│ 폴더 트리 │  ┌─────────────────────────────────────┐  │
│          │  │ 폴더 + 이미지 썸네일                  │  │
│ 폴더 선택 │  │ 또는                                │  │
│ 드래그    │  │ 이미지 뷰어 + 썸네일 스트립            │  │
│ 앤 드롭   │  └─────────────────────────────────────┘  │
└──────────┴───────────────────────────────────────────┘
```

NSSplitViewController가 전체를 관리한다. 왼쪽은 폴더 사이드바. 오른쪽은 브라우저(그리드)와 뷰어(상세)를 전환한다. MainWindowController가 중재자다.

### 의존성 규칙

```
UI (ViewController) → Service → Model
```

역방향 의존은 금지한다. 기능 간 통신은 Delegate 패턴만 사용한다. NotificationCenter나 Combine은 쓰지 않는다. 타입 안전성을 보장하고 추적 가능한 흐름을 유지한다.

### 파일 구조

```
Sources/
├── app/        메인 윈도우, 메뉴, 설정, 도움말
├── sidebar/    폴더 트리 (NSOutlineView)
├── browser/    이미지 그리드 (NSCollectionView)
├── viewer/     이미지 뷰어, 줌, 썸네일 스트립, 동영상
├── services/   파일 관리, 이미지 처리, 캐시, 보안
└── models/     폴더 노드, 이미지 파일, 포맷 정의
```

## 썸네일 캐시: 메모리만, 디스크 없음

1,000장 이상의 이미지를 스크롤할 때 버벅이면 안 된다. 썸네일 캐시가 필요하다.

NSCache 기반 메모리 캐시를 사용한다. 500MB 상한. LRU 정책으로 자동 해제한다. macOS가 메모리 부족을 감지하면 NSCache가 알아서 비운다.

디스크 캐시는 두지 않았다. ImageIO의 썸네일 생성이 충분히 빠르다. 디스크 캐시를 추가하면 무효화 로직이 복잡해진다. 파일이 이동되거나 삭제되면 캐시와 불일치가 생긴다. 그 복잡도를 감수할 만큼 디스크 캐시의 이득이 크지 않았다.

## 줌과 팬: NSScrollView의 magnification

이미지 뷰어는 10%에서 500%까지 확대한다. 확대하면 드래그로 이동한다.

NSScrollView의 `magnification` 프로퍼티를 사용한다. 트랙패드 핀치, Cmd+/-, 키보드 단축키가 모두 이 하나의 프로퍼티를 조작한다. 별도 확대 로직을 구현하지 않았다.

| 단축키 | 동작 |
|---|---|
| Cmd++ | 확대 |
| Cmd+- | 축소 |
| Cmd+0 | 원본 크기 (100%) |
| Cmd+9 | 창에 맞춤 |

## 포맷 변환: ImageIO 양방향

JPEG, PNG, WebP, HEIC, HEIF, AVIF, TIFF, BMP, GIF를 지원한다. 입력과 출력 모두.

변환은 ImageIO 프레임워크가 처리한다. `CGImageSource`로 원본을 읽고, `CGImageDestination`으로 대상 포맷에 쓴다. 외부 라이브러리 없이 시스템 프레임워크만으로 9개 포맷을 처리한다.

## 파일 관리: Finder를 열지 않아도

삭제는 휴지통으로 보낸다. 영구 삭제가 아니다. 이동은 사이드바의 폴더로 드래그한다. 이름 변경은 인라인 편집이다. 클립보드 복사도 된다.

모든 파일 작업은 Result 타입을 반환한다. 실패하면 에러를 명시한다. "권한이 없습니다", "파일이 이미 존재합니다". 사용자에게 실패 이유를 숨기지 않는다.

## 동영상 재생

이미지 뷰어에서 동영상도 재생한다. MP4, MOV, M4V, AVI, MKV. AVKit의 인라인 플레이어를 사용한다. 별도 창을 열지 않는다. 이미지와 동영상이 같은 폴더에 섞여 있어도 자연스럽게 탐색한다.

## 샌드박스와 보안

macOS 앱은 사용자가 허용한 폴더만 접근할 수 있다. 처음 폴더를 선택하면 보안 범위 북마크를 저장한다. 앱을 재시작해도 같은 폴더에 접근할 수 있다.

북마크가 없는 폴더에는 접근하지 못한다. 이것은 제약이 아니다. macOS 보안 모델을 따르는 것이다.

## 빌드: Makefile + swiftc

JenaLab의 macOS 앱 빌드 방식과 동일하다.

```
make run      # 빌드 + 실행
make build    # .app 번들 생성
make install  # ~/Applications에 설치
make dmg      # 배포용 디스크 이미지
make pkg      # 패키지 인스톨러
```

28개 Swift 파일. 외부 의존성 0개. Xcode 프로젝트 0개.

## 정리

이미지 관리에 앱 두 개는 불필요하다. 폴더 트리, 썸네일 그리드, 줌 뷰어, 파일 관리, 포맷 변환. 한 창에 담는다.

NSOutlineView, NSCollectionView, NSScrollView. AppKit의 기본 컴포넌트가 각자의 역할을 한다. 위에 얹은 것은 Delegate 패턴으로 연결한 중재자뿐이다.

---

**JenaImage**는 MIT 오픈소스다.

[GitHub](https://github.com/jenalab-com/jena-image) · [DMG 다운로드](https://github.com/jenalab-com/jena-image/releases/latest/download/JenaImage.dmg)
