# 《천의 얼굴》 — Godot 기술 설계 문서

> 대상: Godot 4.x (2D). Kiro IDE(로컬)에서 이 문서를 기준으로 프로젝트를 생성한다.
> 이 샌드박스(Kiro Web)에서는 Godot를 실행할 수 없으므로, 여기서는 **설계도**만 확정한다.

---

## 1. 왜 Godot 4.x (2D)

- 씬(`.tscn`)·스크립트(`.gd`)가 **텍스트 파일** → 에이전트가 직접 읽고/쓰고/수정 가능.
- GDScript는 파이썬 유사 → 가독성 높고 유지보수 쉬움.
- 2D 턴제 RPG에 가볍고 직관적. 로열티 없음.
- 데이터 주도 설계와 궁합 좋음 (Resource/`.json`/`.tres`).

---

## 2. 프로젝트 구조 (제안)

```
thousand-faces/
├── project.godot
├── docs/                      # 이 설계 문서들을 심볼릭/복사 유지
├── data/                      # 데이터 주도: 책을 끝없이 참조하며 확장
│   ├── chambers/              # Chamber 정의 (JSON 또는 .tres)
│   │   ├── 01_threshold_village.json
│   │   └── ...
│   ├── encounters/            # 조우 정의
│   │   └── ...
│   └── presence_faces.json    # 그분의 얼굴(차원별 묘사/대사/색)
├── scenes/
│   ├── Main.tscn              # 진입점, 상태 머신
│   ├── Chamber.tscn           # 그리드 맵 + 플레이어 이동
│   ├── Encounter.tscn         # 턴제 조우 UI
│   ├── Presence.tscn          # 그분 표현(추상 비주얼)
│   └── ui/
│       ├── ChoicePanel.tscn   # 네 차원 응답 선택
│       └── CoinOfFate.tscn    # 운명의 동전
├── scripts/
│   ├── autoload/
│   │   ├── GameState.gd        # 싱글톤: 4차원 수치, 진행도, 플래그
│   │   ├── DataLoader.gd       # data/ 로드·파싱
│   │   └── AudioDirector.gd    # 앰비언트/차원별 톤
│   ├── chamber/
│   │   ├── Chamber.gd
│   │   └── GridMover.gd        # 그리드 이동 (몰입 유지, 비실시간)
│   ├── encounter/
│   │   ├── Encounter.gd         # 턴 흐름
│   │   ├── Response.gd          # 네 차원 응답 처리
│   │   └── CoinOfFate.gd        # 지배수치 가중 판정
│   └── presence/
│       └── Presence.gd          # 지배 차원→얼굴 렌더
└── assets/
    ├── fonts/
    └── sfx/                     # 초기엔 placeholder
```

---

## 3. 핵심 데이터 모델

### 3.1 GameState (autoload 싱글톤)
```gdscript
# scripts/autoload/GameState.gd
extends Node

# 네 차원 수치 (0~100, UI 비노출)
var dimensions := {
    "awe": 0,     # 경외
    "law": 0,     # 율법
    "mercy": 0,   # 자비
    "logos": 0,   # 앎
}

var flags := {}            # 조우 결과, 닫힌 길(상실의 영구성) 등
var current_chamber := ""

func apply(delta: Dictionary) -> void:
    for k in delta.keys():
        dimensions[k] = clamp(dimensions[k] + delta[k], 0, 100)

func dominant() -> String:
    var best := "awe"
    for k in dimensions.keys():
        if dimensions[k] > dimensions[best]:
            best = k
    return best

# 가로축(자비↔율법), 세로축(경외↔앎) 균형도
func axis_horizontal() -> int:  # +율법 / -자비
    return dimensions["law"] - dimensions["mercy"]
func axis_vertical() -> int:    # +경외 / -앎
    return dimensions["awe"] - dimensions["logos"]
```

### 3.2 Encounter 데이터 (JSON 예시)
```json
{
  "id": "gatekeeper",
  "chamber": "02_guardian_threshold",
  "campbell_stage": "crossing_the_threshold",
  "axis": "horizontal",            // 율법↔자비 딜레마
  "intro": "관문을 지키는 자가 대가를 요구한다.",
  "responses": [
    {
      "dimension": "law",
      "verb": "심판한다",
      "text": "\"규칙은 규칙이다. 너 또한 법 아래 있다.\"",
      "delta": { "law": 12, "mercy": -6 },
      "coin": { "enabled": true, "bias_dim": "law" }
    },
    {
      "dimension": "mercy",
      "verb": "용서한다",
      "text": "\"지나가라. 네 짐을 내가 나눠 지겠다.\"",
      "delta": { "mercy": 12, "law": -6 }
    }
  ]
}
```
> 설계 원칙: 조우 추가 = JSON 파일 추가. **코드 수정 없이 책을 계속 참조하며 확장.**

### 3.3 Presence 얼굴 데이터
```json
{
  "law":   { "color": "#D4A32C", "form": "scales_and_eye",
             "desc": "저울과 눈을 가진 형상이 너를 내려다본다.",
             "mirror_line": "너는 나를 재판관으로 빚었다." },
  "mercy": { "color": "#C65B4E", "form": "enfolding",
             "desc": "상처를 감싸는 둥근 빛이 다가온다.",
             "mirror_line": "너는 용서받고 싶었던 것이다." }
}
```

---

## 4. 상태 머신 (Main.gd)

```
BOOT → CHAMBER(그리드 탐험) → ENCOUNTER(턴제) → RESULT(수치/얼굴)
     → (다음 Chamber 반복) → RETURN → MIRROR(반전) → ENDING
```
- 각 상태는 씬 전환으로 구현. `GameState`는 autoload라 전환에도 유지.

---

## 5. 운명의 동전 판정 로직

```gdscript
# 지배 수치가 높을수록 성공 확률 상승
func resolve_coin(bias_dim: String, player_pick: String) -> bool:
    var bias := float(GameState.dimensions[bias_dim]) / 100.0  # 0~1
    var success_chance := 0.5 + bias * 0.4   # 0.5 ~ 0.9
    var result := "heads" if randf() < success_chance else "tails"
    return result == player_pick
```
- 실패 시 `GameState.flags`에 "닫힌 길" 기록 → 되돌릴 수 없음(F&H 철학).

---

## 6. 비주얼 (추상·미니멀, 에셋 최소)

- 플레이어/그분/맵을 **Polygon2D / Line2D / 셰이더**로 절차적 렌더 → 그림 파일 불필요.
- 차원 색을 전역 테마로 관리. 그분 등장 시 화면 틴트 = 지배 차원 색.
- `CanvasModulate` + `PointLight2D`로 Look Outside식 암전/조명.

---

## 7. Kiro IDE로 넘어간 뒤 첫 작업 순서

1. Godot 4.x 설치 확인, 빈 2D 프로젝트 생성 (`project.godot`).
2. `GameState`, `DataLoader` autoload 작성.
3. `Chamber.tscn` + 그리드 이동 (Chamber 1: 문턱의 마을).
4. `Encounter.tscn` + 네 차원 응답 (조우 1: 관문의 수호자, 율법↔자비).
5. `Presence.tscn` 얼굴 1종 변화 확인.
6. → **수직 슬라이스 완성 → 플레이로 검증.**

> 이 샌드박스에서는 1~6을 실행 확인할 수 없다. Kiro IDE(로컬)에서 실행하며 진행.
