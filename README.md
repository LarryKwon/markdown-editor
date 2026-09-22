# 마크다운 노트

여러 사람이 함께 쓰는 마크다운 노트 페이지. 배포판이 두 가지입니다.

| 파일 | 배포처 | 동기화 방식 |
| --- | --- | --- |
| `index.html` | Claude Artifact | 아티팩트 공유 데이터베이스 (서버 저장, 영구) |
| `docs/index.html` | GitHub Pages | Yjs + WebRTC P2P (서버 없음, 링크 기반) |

두 빌드는 UI와 CSS를 공유하고 데이터 계층만 다릅니다.
`docs/index.html`의 스타일은 `index.html`의 `<style>` 블록을 그대로 가져온 것이므로,
디자인을 고칠 때는 `index.html`을 먼저 고치고 `docs/index.html`에 반영하세요.

## Artifact 빌드 (`index.html`) 동작 방식

- **저장**: 입력이 멈추고 0.6초 뒤 자동 저장 (`Ctrl`/`Cmd` + `S`로 즉시 저장).
- **동기화**: 노트는 아티팩트의 공유 데이터베이스(`db` capability)의 `notes`
  컬렉션에 문서 하나씩 저장되고, `onSnapshot` 구독으로 모든 화면에 실시간 반영됩니다.
- **충돌 처리**: 마지막 저장이 이깁니다(last-writer-wins). 내가 입력하는 중에
  다른 사람이 저장하면 알림 배너를 띄워 최신 내용을 불러올 수 있게 합니다.
- **접속자 표시**: `room` capability의 presence로 지금 페이지를 보는 사람과
  각자 보고 있는 노트를 표시합니다. 이름/아바타는 `user` capability로 조회합니다.
- **권한**: 편집 권한이 없는 뷰어는 읽기 전용으로 전환됩니다.
- **오프라인/미지원 환경**: 공유 저장소를 쓸 수 없으면 localStorage에만 저장하고
  상태 표시에 "이 브라우저에만 저장됨"을 노출합니다.

## 노트 문서 형태

```json
{
  "title": "첫 줄에서 뽑은 제목",
  "body": "# 마크다운 원문",
  "updatedAt": 1790035236723,
  "updatedBy": "u_...",
  "createdAt": 1790035236723
}
```

## 렌더링

마크다운은 marked 4.3.0으로 변환하고 DOMPurify 3.0.6으로 살균합니다.
(다른 사람이 쓴 내용은 신뢰할 수 없는 입력이므로 항상 살균 후 삽입합니다.)
라이브러리를 불러오지 못하면 원문을 그대로 `<pre>`로 보여줍니다.

## 배포

Claude Artifact로 게시합니다. capability 선언:

```json
{"db": {}, "user": {"scopes": ["profile"]}, "room": {}}
```


## GitHub Pages 빌드 (`docs/index.html`)

- **동기화**: 같은 링크(`#r=<방>&k=<열쇠>`)를 연 브라우저끼리 WebRTC로 직접 연결됩니다.
  공개 시그널링 서버(`wss://y-webrtc-eu.fly.dev`)는 서로를 찾아주는 역할만 하고,
  노트 내용은 `k` 값을 열쇠로 암호화되어 피어 사이에서만 오갑니다.
- **문서 모델**: Yjs CRDT. 본문은 `Y.Text`라 같은 문장을 동시에 고쳐도 글자 단위로 병합됩니다
  (Artifact 빌드의 last-writer-wins보다 강합니다).
- **저장**: `y-indexeddb`로 각자 브라우저에 보관됩니다. 접속자가 없을 때의 변경은
  각 브라우저에 남아 있다가 다시 만나면 합쳐집니다.
- **주의**: 링크를 가진 사람은 누구나 읽고 쓸 수 있습니다. 링크 자체가 열쇠입니다.
  또한 처음 들어온 사람은 다른 참여자가 접속하기 전까지 내용이 비어 보일 수 있습니다.
- **폴백**: 모듈(esm.sh)을 불러오지 못하면 배너를 띄우고 localStorage 전용 모드로 동작합니다.
  이 모드에서 쓴 노트는 나중에 P2P 문서로 자동 이전되지 않습니다.

### 배포 설정

GitHub Settings → Pages → Source를 `Deploy from a branch`,
브랜치 `main`, 폴더 `/docs`로 지정하면 `https://<사용자>.github.io/markdown-editor/`로 게시됩니다.
