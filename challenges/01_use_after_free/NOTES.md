# 01_use_after_free 학습 노트

## 진행 상황 (2026-09-28)

`main` 함수를 한 줄씩 해석하는 중이다. 지금까지 **149번째 줄 `Screen s = { .count = 0 };`** 을 이해했다.

### 1. 설계도와 실행은 다르다
- `#define`, `typedef struct`, `struct` 는 **실행되지 않는 설계도**다. 컴파일러가 컴파일할 때 읽는다.
- 실제로 실행되는 건 `main` 안의 코드뿐이고, 위에서 아래로 한 줄씩 진행한다.
- 설계도를 `Screen → MAX_WIDGETS → struct Widget → VTable` 순서로 따라가는 건 **읽는 순서**이지 실행 순서가 아니다.

### 2. `Screen s = { .count = 0 };` 해석
- `main`의 스택에 `Screen` 변수 `s`가 하나 생긴다.
- `s` 안에는 다음 두 가지만 있다.
  - `Widget *items[8]`: `Widget`을 **가리키는 포인터(주소) 8칸**, `items[0]` ~ `items[7]` (0~8이 아님)
  - `int count`
- `.count = 0` 은 **`count`만** 0으로 지정한다.
- 적지 않은 멤버(`items`)는 C 규칙에 따라 **자동으로 0(NULL)** 이 된다.
- 결과: `items[0]~[7]` 은 모두 `NULL`, `count` 는 0. **`Widget` 본체는 아직 하나도 없다.**

```
s (스택, 72바이트)
┌───────────────┐
│ items[0] NULL │  ← 주소 칸만 8개 (각 8바이트)
│ ...           │
│ items[7] NULL │
│ count = 0     │
└───────────────┘
힙: Widget 없음
```

### 3. 정정한 오해
- ❌ `items` 안에 `Widget`(vtbl, id, closed, label) 8개가 들어 있다
  → ✅ `items` 에는 **주소만** 들어간다. `Widget` 본체는 나중에 `widget_new()` 가 힙에 따로 만든다.
- ❌ `s` 를 만들 때 `struct Widget`, `VTable` 설계도도 참고한다
  → ✅ **`Screen` 설계도만** 있으면 된다. 포인터는 무엇을 가리키든 크기가 8바이트라서 `Widget` 의 내용과 상관없다.
- ❌ `.count = 0` 은 모든 값을 0으로 만든다
  → ✅ `count` 만 지정한다. 나머지가 0이 되는 건 별도 규칙이다. (`{ .count = 5 }` 라면 `count` 는 5, `items` 는 NULL)

### 4. 곁가지로 배운 것
- 배열 초기화는 `.` 과 중괄호를 쓴다: `{ .items = { a } }`, `{ .items[3] = a }`
  - 칸 수(8) 이하라면 몇 개를 넣어도 되고, 적지 않은 칸은 NULL이 된다.
- 선언이 끝난 뒤에는 `s.items = { ... }` 처럼 통째로 대입할 수 없다. `s.items[0] = a;` 처럼 한 칸씩 넣어야 한다.
- `{ .items = 1 }` 처럼 정수를 포인터 칸에 넣으면 gcc 13은 경고(`-Wint-conversion`), clang은 에러를 낸다.
  - 실행하면 주소 1번지를 읽다가 SIGSEGV가 난다.
  - 예외로 `0` 은 널 포인터로 인정되어 괜찮다.

### 각 설계도가 쓰이는 곳
| 코드 줄 | 필요한 설계도 |
|---|---|
| 149 `Screen s = { .count = 0 };` | `Screen` 만 |
| 151~154 `widget_new` / `screen_add` | `struct Widget` (vtbl, id, closed, label) |
| 157 이후 `screen_render` / `screen_dispatch` | `VTable` (render, on_event) |

## 다음에 할 것

**151번째 줄부터** 이어서 해석한다.

```c
screen_add(&s, widget_new(&LABEL_VT, 10, "Welcome"));
```

1. 함수 안에 함수가 있으면 **안쪽이 먼저** 실행된다. 그래서 `widget_new` → `screen_add` 순서로 읽는다.
2. `widget_new` (85~102번째 줄): `Widget` 을 힙에 어떻게 만들고 무엇을 돌려주는지 해석한다.
3. `screen_add` (109~111번째 줄): 돌려받은 주소가 `items[]` 에 어떻게 들어가는지 해석한다.
4. 이후 152~154번째 줄 → `screen_render` → `screen_dispatch` → `dialog_on_event` 순서로 진행해서 UAF가 생기는 지점을 찾는다.
