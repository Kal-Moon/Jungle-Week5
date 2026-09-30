# 01_use_after_free 학습 노트

> 줄 번호는 로컬 파일마다 다를 수 있어서 **코드 내용이나 함수 이름**으로 적는다.

## 진행 기록

| 날짜 | 한 것 |
|---|---|
| 2026-09-28 | `Screen s = { .count = 0 };` 해석 |
| 2026-09-29 | `widget_new`, `screen_add`, `screen_render`, `screen_dispatch`, `dialog_on_event`, `app_build_status`, frame 2 크래시 원인까지 해석 |
| 2026-09-30 | `malloc`/`free` 재사용 과정을 단계별로 다시 이해하고 **버그 수정 완료** (8~9장) |

## 상태: ✅ 해결 완료

수정한 `bug.c`는 경고 없이 빌드되고, 기대 출력대로 정상 종료(0)한다. AddressSanitizer/UBSan 검사에서도 오류가 없다.

## 다음에 할 것 (여기서부터 시작)

1. (선택) 수정한 `bug.c`에서 코드와 맞지 않게 된 주석 정리
   - `dialog_on_event` 선언 위 `/* 다이얼로그는 ... 스스로 정리(파괴)된다 */` → 이제는 `closed` 표시만 하고 정리는 `main`이 한다.
2. **다음 문제 `02_stack_buffer_overflow`** 시작. 01번과 같은 방식으로 진행한다.
   - 파일 맨 위 주석의 `[시나리오]`, `[증상]`, `TODO` 먼저 읽기
   - `main`부터 한 줄씩 해석 → 함수끼리의 상호작용 → 크래시 원인 → 수정

---

## 0. 기본 개념
- **설계도는 실행되지 않는다.** `#define`, `typedef struct`, `struct`는 컴파일러가 읽는 설계도다. 실행되는 건 `main` 안의 코드뿐이다.
- **함수 안에 함수가 있으면 안쪽이 먼저** 실행된다.
- **선언**은 새 변수를 만드는 것(`Widget *w = ...;`), **대입**은 이미 있는 칸에 값을 넣는 것(`self->closed = 1;`)이다.
- 함수 선언을 읽는 법: 맨 앞은 **돌려주는 것**, 이름 뒤 `( )` 안은 **받는 것**.
- `static` 함수는 "이 파일 안에서만 쓰는 함수"라는 뜻이다. 메모리 위치와는 관계없다.
- `exit(1)`은 함수만 빠져나가는 게 아니라 **프로그램 전체를 종료**한다.

## 1. `Screen s = { .count = 0 };`
- `main`의 스택에 `s`가 생긴다. 안에는 `Widget *items[8]`(주소 8칸, `items[0]`~`items[7]`)과 `int count`만 있다.
- `.count = 0`은 `count`만 지정한다. 적지 않은 `items`는 자동으로 `NULL`이 된다.
- `Widget` 본체는 아직 없다. `s`를 만들 때는 `Screen` 설계도만 필요하다.

## 2. `screen_add(&s, widget_new(&LABEL_VT, 10, "Welcome"));`
`widget_new`가 먼저 실행되고, 돌려준 주소를 `screen_add`가 받는다.

### `widget_new(vt, id, label)`
| 코드 | 하는 일 |
|---|---|
| `Widget *w = malloc(sizeof *w);` | 힙에서 `Widget` 크기(40바이트)를 받는다 |
| `if (!w) { perror(...); exit(1); }` | 실패(`NULL`)하면 프로그램 종료 |
| `w->vtbl = vt;` `w->id = id;` `w->closed = 0;` | 칸 채우기 |
| `strncpy(w->label, label, 23);` | 최대 23글자 복사, 짧으면 나머지는 `\0` |
| `w->label[23] = '\0';` | 배열 마지막 칸에 `\0` (긴 문자열 대비 안전장치) |
| `return w;` | 만든 `Widget`의 **주소**를 돌려준다 |

### `screen_add(s, w)`
```c
if (s->count < MAX_WIDGETS) s->items[s->count++] = w;
```
- `items[count]`에 주소 `w`를 넣고, 그다음 `count`를 1 올린다(`count++`는 지금 값을 먼저 쓴다).
- `count`는 "채운 칸 수"이자 "다음에 채울 칸 번호". 0부터 최대 8까지.

### 네 줄 실행 후 (`widget_new` → `screen_add`를 한 줄씩 짝지어 실행)
```
s (스택)                         힙
│ items[0] ──▶ Widget{ &LABEL_VT,  10, "Welcome" }
│ items[1] ──▶ Widget{ &BUTTON_VT, 11, "OK" }
│ items[2] ──▶ Widget{ &DIALOG_VT, 12, "Are you sure?" }
│ items[3] ──▶ Widget{ &BUTTON_VT, 13, "Cancel" }
│ items[4]~[7] NULL
│ count = 4
```

## 3. `VTable`과 함수 포인터
- `void (*render)(Widget *self);` → `render`는 `Widget *`를 받고 아무것도 돌려주지 않는 **함수를 가리키는 포인터**.
- `void (*on_event)(Widget *self, int code);` → `Widget *`와 `int`를 받는 함수를 가리키는 포인터.

| 선언 | 정체 |
|---|---|
| `void (*render)(Widget *self);` | 함수를 가리키는 포인터 |
| `void render(Widget *self);` | 함수 |
| `void *render(Widget *self);` | `void *`를 돌려주는 함수 (예: `malloc`) |

```c
LABEL_VT  = { label_render,  widget_noop_event };
BUTTON_VT = { button_render, widget_noop_event };
DIALOG_VT = { dialog_render, dialog_on_event  };
```

## 4. `screen_render(&s);` (frame 1)
```c
for (int i = 0; i < s->count; i++) {
    Widget *w = s->items[i];
    w->vtbl->render(w);
}
```
- `i`는 0~3, **`count`번** 돈다. `w`는 하나이고 매번 `items[i]`의 주소가 복사돼 들어간다.
- `w->vtbl->render(w)`: `Widget` → `VTable` → 거기 적힌 `render` 함수 호출. 위젯마다 다른 함수가 불린다.

## 5. `screen_dispatch(&s, 1);`
`screen_render`와 모양이 같고 마지막 줄만 `w->vtbl->on_event(w, code);`. Label/Button은 `widget_noop_event`(아무것도 안 함), **Dialog만 `dialog_on_event(w, 1)`**.

```c
// dialog_on_event
if (code == 1) {
    self->closed = 1;       // 닫힘 표시 대입 (지금은 아무도 확인하지 않음)
    widget_destroy(self);   // free(self): 메모리 반납
}
```
⚠️ `free`는 메모리만 반납하고 **`s.items[2]`의 주소는 그대로 남긴다** → **댕글링 포인터**. `dialog_on_event`는 `s`를 모르니 지울 수도 없다.

## 6. `char *status = app_build_status("dialog closed");`
| 코드 | 하는 일 |
|---|---|
| `char *msg = malloc(sizeof(Widget));` | 40바이트를 받는다 |
| `if (!msg) exit(1);` | 실패하면 프로그램 종료 |
| `memset(msg, 0xAB, sizeof(Widget));` | `msg`부터 40바이트를 `0xAB`로 채운다 (어디서, 무슨 값, 몇 바이트) |
| `snprintf(msg, sizeof(Widget), "STATUS: %s", text);` | `msg`에 최대 40바이트로 `"STATUS: dialog closed"`를 쓴다 (`%s` ← `text`) |
| `return msg;` | 주소를 돌려준다 |

**핵심:** 방금 반납된 Dialog와 **같은 크기(40바이트)**를 요청해서, `malloc`이 **그 칸을 그대로 다시 빌려준다.** (`[테스트용 연출]` 주석 참고, glibc 환경)

## 7. frame 2의 `screen_render(&s);`, `i` = 2일 때 → 크래시
주소를 찍는 복사본으로 실행한 결과:
```
[render] i=2 w=0x...6300 vtbl=0x...ad60           ← frame 1: DIALOG_VT
[destroy]    w=0x...6300 vtbl=0x...ad60           ← dispatch: free
[status]   msg=0x...6300                          ← 같은 주소를 다시 받음!
[render] i=2 w=0x...6300 vtbl=0x203a535554415453  ← vtbl 망가짐
Segmentation fault
```
- `w`(= `items[2]`)의 **주소는 그대로**다. 하지만 그 주소의 **내용이 바뀌었다.**
- `vtbl` 자리(0~7바이트)에 `"STATUS: "` 8글자가 들어갔다. `0x203a535554415453`을 거꾸로 읽으면 `S T A T U S : (공백)`.
  ```
  바이트    0~7        8~11   12~15   16~39
  Widget:  vtbl        id     closed  label
  실제:    "STATUS: "  "dial" "og c"  "losed" \0 AB AB ...
  ```
- 존재하지 않는 주소에서 `render`를 읽으려다 **SIGSEGV**.
- 화면에 frame 1 출력이 안 보이는 건 `printf`(stdout)가 버퍼에 쌓여 있다가 크래시로 사라졌기 때문. 로그는 `stderr`로 찍어야 한다.

> **크래시는 `screen_render`에서 나지만, 원인은 `dialog_on_event`가 `free`만 하고 `items[2]`를 그대로 둔 것이다.**

## 8. 단계별로 다시 이해한 것 (사물함 비유)

힙 = 사물함 보관소. `malloc` = 빌리기(사물함 **번호**=주소를 받음), `free` = 반납(장부에 "비어 있음"이라고 적을 뿐, 번호를 적어둔 곳은 그대로).

1. **빌림**: 세 번째 `screen_add`의 `widget_new`가 `malloc`으로 6300번을 빌림 → `items[2] = 6300`
2. **사용**: frame 1 `screen_render` → 6300번은 아직 Dialog → 정상
3. **반납**: `screen_dispatch` → (`vtbl`의 `on_event` 칸을 통해) `dialog_on_event` → `widget_destroy` → `free(6300)`. **`items[2]`에는 6300이 그대로**
4. **남이 가져감**: `app_build_status`의 `malloc(40)`이 방금 반납된 **같은 크기** 칸을 다시 줌 → `msg = status = 6300`
   - 이제 `status`와 `items[2]`가 같은 사물함을 각자 다른 용도로 쓴다고 믿는 상태
5. **반납한 걸 또 씀**: frame 2 `screen_render`가 `i = 0`부터 돌다가 `i = 2`에서 6300번을 Dialog로 믿고 사용 → 💥

헷갈렸던 점 정리:
- `DIALOG_VT`는 `screen_add` 때 **실행되지 않는다.** `vtbl`에 "연락처 카드"(주소)를 붙여둘 뿐이고, `screen_render`/`screen_dispatch`가 카드를 보고 호출할 때 실행된다.
- `screen_dispatch`가 직접 `free`하는 게 아니라, 불린 함수의 사슬 끝(`widget_destroy`)에서 `free`한다.
- 10, 11, 13은 **반납하지 않았으니** 계속 써도 안전하다. 문제는 `free`를 안 해서가 아니라 **`free`한 뒤에 또 써서** 생긴다.
- Dialog를 마지막까지 안 지우면 크래시는 안 나지만, frame 2에도 Dialog가 그려진다(기대 동작 위반). 안 쓰는 메모리는 바로 반납하는 게 원칙이다(안 하면 메모리 누수).
- frame은 창이 아니라 "화면을 한 번 그린 것". 터미널에는 frame 1, 2가 위아래로 이어서 출력된다.
- `status`는 닫는 일을 하지 않는다. 닫힌 뒤 "dialog closed" 안내 문구를 만들 뿐이다.

## 9. 수정 (해결)

역할을 나눴다: **표시는 위젯이, 정리(`free` + `NULL`)는 `s`를 아는 `main`이, 사용하는 쪽은 `NULL`을 건너뛴다.**

| 위치 | 수정 |
|---|---|
| `dialog_on_event` | `widget_destroy(self);` 삭제, `closed = 1`만 남김 |
| `main`의 TODO 자리 (`screen_dispatch` 뒤, `app_build_status` 앞) | `closed`인 위젯을 `free` + 그 칸을 `NULL` |
| `screen_render`, `screen_dispatch` | `if (w == NULL) continue;` |
| `widget_destroy` | 쓰는 곳이 없어져서 `-Wunused-function` 경고 → 주석 처리로 제거 |

```c
static void dialog_on_event(Widget *self, int code) {
    if (code == 1) {
        self->closed = 1;
    }
}

// screen_render / screen_dispatch 반복문 안
Widget *w = s->items[i];
if (w == NULL) continue;

// main: screen_dispatch(&s, 1); 바로 뒤
for (int i = 0; i < s.count; i++){
    Widget *w = s.items[i];
    if(w->closed == 1){
        free(w);
        s.items[i] = NULL;
    }
}
```

수정 후 출력 (종료 코드 0):
```
frame 1:
  Label #10: Welcome
  [Button #11] "OK"
  <<Dialog #12>> Are you sure?
  [Button #13] "Cancel"
STATUS: dialog closed
frame 2:
  Label #10: Welcome
  [Button #11] "OK"
  [Button #13] "Cancel"
```

### 수정하면서 배운 것
- **`continue` vs `break`**: `continue`는 이번 칸만 건너뛰고 다음 `i`로, `break`는 반복문 전체 종료. `break`를 쓰면 frame 2에 Label 10, Button 11만 그려지고 Cancel이 빠진다.
- C에는 `next` 명령어가 없다. `w->next`는 `w`의 `next` 칸을 읽는 것이라, `w`가 `NULL`이면 오히려 크래시.
- **`NULL`을 읽으면(널 포인터 역참조) SIGSEGV.** `NULL`로 지우는 것(반납하는 쪽)과 `NULL`을 건너뛰는 것(사용하는 쪽)을 **둘 다** 해야 한다.
- `screen_dispatch`는 지금 `main`에서는 `NULL`이 생기기 전에만 불리지만, 이벤트가 또 오면 크래시하므로 검사가 필요하다. 원칙: **`NULL`이 될 수 있는 칸을 꺼내 쓰는 곳은 전부 검사.**
- (선택) 정리 반복문 조건을 `w != NULL && w->closed == 1`로 쓰면 여러 번 실행해도 안전하다. `&&`는 왼쪽이 거짓이면 오른쪽을 실행하지 않는다.
- **`free(NULL)`은 아무 일도 안 한다**(C 표준). 그래서 마지막 `free(s.items[i])` 반복문은 그대로 둬도 된다.
- 수정 전이었다면 마지막에 `free(status)`와 `free(items[2])`가 **같은 6300번을 또 반납** → **Double Free**(`04_double_free` 주제). `NULL`로 지우는 것 하나로 UAF와 Double Free를 둘 다 막았다.
- `widget_destroy`는 지워도 되지만, 남겨두면 위젯 구조가 바뀌어도 한 곳만 고치면 되고 gdb에서 `break widget_destroy`로 잡을 수 있다는 장점이 있다(`widget_new`와 짝).

> **"해제 = 소유 포인터 무효화"**: 반납했으면, 그 번호를 적어둔 곳도 같이 지운다.
