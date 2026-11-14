# CLAUDE.md

이 레포에서 작업하는 AI를 위한 가이드. 목표와 로드맵은 [PRD.md](PRD.md) 참고.

## 프로젝트 개요

인크리멘탈 게임용 업그레이드 트리 에디터. **빌드 없는 단일 HTML 파일**(`index.html`)이며 바닐라 JS + SVG로 동작한다.

## 구조

```
index.html                 # 앱 전체 (CSS + HTML + JS)
examples/stranded-god.json # 예시 데이터
README.md / PRD.md / CLAUDE.md
```

`index.html`의 `<script>`는 주석 구획으로 나뉜다:

| 구획 | 내용 |
|---|---|
| 데이터 | `SCHEMA`, `sample()`, `normalize()` — 스키마의 기준 |
| 상태 / 저장 / 실행취소 | 전역 `P`(프로젝트), `sel`, localStorage, 히스토리 |
| 검증 | `validate()` → `{lv:'e'|'w'|'i', msg, id}` 배열 |
| 레이아웃 | `layout()` — 의존 깊이 = x, 시대 순서 + 부모 무게중심 = y |
| 그래프 렌더 | `renderGraph()`, 팬/줌 |
| 왼쪽 / 오른쪽 패널 | 목록·재화·시대·프로젝트 탭 / 노드 편집 |
| 모달 | 내보내기, AI 마크다운(`aiMarkdown()`), 불러오기 + `diff()` |

## 작업 규칙

- **외부 의존성·빌드 도구 추가 금지.** 파일 하나를 브라우저로 열면 동작해야 한다. 꼭 필요하면 CDN 스크립트 1개까지만, 먼저 제안할 것.
- 데이터 변경 후에는 `changed()`를 호출한다 (저장 + 재렌더 + 실행취소 스냅샷). 구조가 바뀌면 `changed(true)` 후 `renderAll()`.
- 오른쪽 편집 패널의 입력은 `data-p="경로"` + `data-t="num|bool|numOrStr|tags"`로 자동 바인딩된다. 새 필드도 이 방식을 따른다.
- 입력 중 포커스가 날아가지 않도록, 타이핑 이벤트에서는 편집 패널 전체를 다시 그리지 않는다 (`renderDerived()`만).
- UI 문구는 한국어.
- 수정 후 브라우저에서 열어 콘솔 오류가 없는지 확인한다.

## 데이터 스키마 (`upgrade-tree/1`)

```jsonc
{
  "schema": "upgrade-tree/1",
  "meta": { "name": "", "concept": "" },
  "resources": [{ "id": "wood", "name": "나무", "manual": true, "note": "" }],
  "eras": [{ "id": "primitive", "name": "원시", "color": "#f2a541" }],
  "nodes": [{
    "id": "stone_axe",              // snake_case, 고유, 참조 키
    "name": "돌도끼",
    "era": "primitive",
    "status": "idea",               // idea | wip | done | cut
    "maxLevel": 5,                  // 1 = 1회성
    "cost": [{ "resource": "wood", "base": 10, "growth": 1.15 }],  // ceil(base × growth^(lv−1))
    "requires": {
      "all": [{ "node": "fire", "minLevel": 1 }],      // AND
      "any": [{ "resource": "stone", "reached": 50 },  // OR (비어 있으면 무시)
              { "custom": "자유 텍스트 조건" }]
    },
    "effects": [{ "type": "multiply", "target": "wood.production", "value": 1.5, "perLevel": true }],
    // type: produce | multiply | add | unlock | custom
    "description": "나무 생산량 ×{value1}",  // {valueN} = effects[N-1].value, {maxLevel}
    "designNote": "",
    "tags": []
  }]
}
```

### 스키마 변경 규칙

- 필드를 추가하면 `normalize()`에 기본값을 넣어 **옛 JSON도 그대로 열리게** 한다.
- 호환되지 않는 변경이면 `SCHEMA` 버전을 올리고 `normalize()`에 마이그레이션을 추가한다.
- `aiMarkdown()`의 스키마 설명과 이 문서도 함께 갱신한다.

## 게임 데이터 작업 요청을 받았을 때

사용자가 이 툴에서 내보낸 마크다운을 붙여넣고 노드 추가·수정을 요청하면:

1. 위 스키마 그대로 **전체 JSON**을 ```json 코드블록 하나로 반환
2. 기존 노드 `id`는 바꾸지 않음
3. 수치는 `effects`에 구조화하고 `description`은 `{value1}` 템플릿 사용
4. 새로 추가하거나 크게 바꾼 노드는 `designNote`에 의도 기록
5. JSON 뒤에 변경 요약
