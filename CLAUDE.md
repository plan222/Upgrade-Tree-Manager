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
| 레이아웃 | `layout()` — 의존 깊이 = x, 시대 순서 + 부모 무게중심 = y. 노드에 `pos`가 있으면 그 값이 자동 위치를 덮어씀 |
| 그래프 렌더 | `colsSVG()`/`edgesSVG()`/`nodesSVG()` → `#cols`/`#edges`/`#nodes` 그룹에 각각 그림. `renderGraph()`가 전체, 드래그 중에는 `edgesSVG()`만 다시 그림 |
| 그래프 상호작용 | `graphEvents()` — 빈 곳 드래그 = 팬, 노드 드래그 = `pos` 고정, 포트(`.port`) 드래그 = `addDep()`로 의존성 연결 |
| 왼쪽 / 오른쪽 패널 | 목록·재화·시대·프로젝트 탭 / 노드 편집(`renderRight()`) · 다중 선택 일괄 편집(`renderBulk()`) |
| 비용 곡선 | `openCurve()` / `curveSVG()` — 인라인 SVG 선 그래프 |
| 로컬 파일 | File System Access API. `saveFile()`/`saveFileAs()`/`openFilePicker()`, 핸들은 IndexedDB(`idbSet/idbGet`)에 보관, 상태 표시는 `renderFile()` |
| 모달 | 내보내기, AI 마크다운(`aiMarkdown()`), 불러오기 + `diff()` + `mergePick()`(선택 적용) |

## 작업 규칙

- **외부 의존성·빌드 도구 추가 금지.** 파일 하나를 브라우저로 열면 동작해야 한다. 꼭 필요하면 CDN 스크립트 1개까지만, 먼저 제안할 것.
- 데이터 변경 후에는 `changed()`를 호출한다 (localStorage 저장 + 파일 dirty 표시 + 재렌더 + 실행취소 스냅샷). 구조가 바뀌면 `changed(true)` 후 `renderAll()`.
- 선택 상태는 `sel`(주 선택) + `selSet`(다중 선택) 두 개다. 직접 건드리지 말고 `select(id, additive)`를 쓰고, 노드가 사라질 수 있는 작업 뒤에는 `pruneSel()`을 호출한다.
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
    "tags": [],
    "pos": null                     // 수동 배치 좌표 {x,y} 또는 null(자동 정렬)
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
3. 수치는 `effects`에 구조화하고 `description`은 `{value1}` 템플릿 사용. 기존 `pos` 값은 그대로 두고, 새 노드는 `pos: null`
4. 새로 추가하거나 크게 바꾼 노드는 `designNote`에 의도 기록
5. JSON 뒤에 변경 요약
