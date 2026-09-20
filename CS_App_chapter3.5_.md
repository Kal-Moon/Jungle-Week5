3.5 산술연산과 논리연산
인스트럭션 "클래스"라는 개념

x86-64의 산술 인스트럭션은 오퍼랜드 길이에 따라 4가지 변형을 갖습니다. 예를 들어 ADD 클래스는 실제로는 네 개의 인스트럭션입니다.

인스트럭션	오퍼랜드 크기
addb	바이트(1)
addw	워드(2)
addl	더블워드(4)
addq	쿼드워드(8)

그래서 책에서는 매번 네 개를 나열하는 대신 ADD S, D처럼 대문자 클래스 이름으로 표기합니다. 연산은 네 그룹으로 나뉩니다: 유효주소 적재 / 단항 / 이항 / 쉬프트. 이항 연산은 오퍼랜드가 둘, 단항 연산은 하나입니다. (leaq만은 길이 변형이 없습니다.)

그림 3.10 — 정수 산술 및 논리연산
Instruction	Effect	설명
leaq S, D	D ← &S	유효주소 적재
INC D	D ← D + 1	증가
DEC D	D ← D − 1	감소
NEG D	D ← −D	부호 반전
NOT D	D ← ~D	보수
ADD S, D	D ← D + S	덧셈
SUB S, D	D ← D − S	뺄셈
IMUL S, D	D ← D * S	곱셈
XOR S, D	D ← D ^ S	배타적 논리합
OR S, D	D ← D | S	논리합
AND S, D	D ← D & S	논리곱
SAL k, D	D ← D << k	좌측 쉬프트
SHL k, D	D ← D << k	좌측 쉬프트 (SAL과 동일)
SAR k, D	D ← D >>ᴀ k	산술 우측 쉬프트
SHR k, D	D ← D >>ʟ k	논리 우측 쉬프트

ATT 포맷에서는 소스가 먼저, 목적지가 나중입니다. subq %rax, %rdx는 "%rdx에서 %rax를 뺀다"(D ← D − S)입니다. 뺄셈처럼 교환법칙이 성립하지 않는 연산에서 매번 헷갈리는 지점이니, "뒤에 오는 게 누적되는 쪽"으로 외워두면 편합니다.

3.5.1 유효주소 적재 (leaq)

leaq는 겉보기엔 메모리에서 레지스터로 읽어오는 형태지만 메모리를 전혀 참조하지 않습니다. 가리키는 위치에서 값을 읽는 대신, 계산된 유효주소 자체를 목적지에 복사합니다. C의 & 연산자에 대응합니다.

용도는 두 가지입니다.

나중에 쓸 포인터를 만들 때 (원래 목적)
간단한 산술 계산을 한 인스트럭션으로 압축할 때 (컴파일러가 훨씬 자주 쓰는 용도)

%rdx에 x가 들어 있을 때 leaq 7(%rdx,%rdx,4), %rdx는 %rdx에 5x + 7을 저장합니다. 주소 계산과 아무 상관이 없어도 컴파일러는 이 형태를 즐겨 씁니다.

long scale(long x, long y, long z) { return x + 4*y + 12*z; }

  x in %rdi, y in %rsi, z in %rdx
scale:
  leaq (%rdi,%rsi,4), %rax    # x + 4*y
  leaq (%rdx,%rdx,2), %rdx    # z + 2*z = 3*z
  leaq (%rax,%rdx,4), %rax    # (x+4*y) + 4*(3z) = x + 4*y + 12*z
  ret

덧셈과 (2·3·4·5·8·9 같은) 제한된 형태의 곱셈을 한 번에 처리하는 능력이 이렇게 쓰입니다.

💡 leaq의 목적 오퍼랜드는 반드시 레지스터입니다. 그리고 뒤에서 중요해지는 성질 하나 — leaq는 조건 코드를 변경하지 않습니다.

3.5.2 단항 및 이항 연산
단항: 오퍼랜드 하나가 소스이자 목적지. 레지스터든 메모리든 가능합니다. incq (%rsp)는 스택 탑의 8바이트 값을 1 증가시킵니다. C의 ++, --를 연상하면 됩니다.
이항: 두 번째 오퍼랜드가 소스이자 목적지. C의 x -= y와 같은 꼴입니다. 첫 번째 오퍼랜드는 상수/레지스터/메모리, 두 번째는 레지스터/메모리.

MOV와 마찬가지로 두 오퍼랜드가 모두 메모리일 수는 없습니다. 두 번째 오퍼랜드가 메모리면 프로세서는 읽고 → 연산하고 → 다시 쓰는 세 동작을 수행합니다.

3.5.3 쉬프트 연산

쉬프트 양을 먼저, 쉬프트할 값을 나중에 줍니다. 쉬프트 양은 즉시 값이거나 단일 바이트 레지스터 %cl 로만 줄 수 있습니다(이 레지스터만 허용된다는 점이 특이합니다).

여기서 시험에 잘 나오는 규칙 하나. w비트 데이터에 대한 쉬프트는 %cl의 하위 m비트만 사용하며, 이때 2^m = w입니다. 상위 비트는 무시됩니다.

인스트럭션	데이터 폭 w	사용하는 비트 수 m	%cl이 0xFF일 때 실제 쉬프트 양
salb	8	3	7
salw	16	4	15
sall	32	5	31
salq	64	6	63
SAL과 SHL은 이름만 둘이고 효과는 같습니다(우측에서 0을 채움).
우측은 다릅니다. SHR은 논리 쉬프트(0으로 채움), SAR은 산술 쉬프트(부호 비트를 복사해 채움).
3.5.4 토의 — 왜 2의 보수인가

그림 3.10의 거의 모든 인스트럭션은 비부호형과 2의 보수 산술 양쪽에 그대로 쓰입니다. 부호형/비부호형을 구분해야 하는 것은 우측 쉬프트뿐입니다(그리고 뒤에 나오는 나눗셈·완전 곱셈). 이것이 부호형 정수를 2의 보수로 표현하는 가장 큰 실용적 이유입니다 — 하드웨어를 두 벌 만들 필요가 없습니다.

long arith(long x, long y, long z) {
    long t1 = x ^ y;
    long t2 = z * 48;
    long t3 = t1 & 0x0F0F0F0F;
    long t4 = t2 - t3;
    return t4;
}

  x in %rdi, y in %rsi, z in %rdx
arith:
  xorq %rsi, %rdi            # t1 = x ^ y
  leaq (%rdx,%rdx,2), %rax   # 3*z
  salq $4, %rax              # t2 = 16 * (3*z) = 48*z
  andl $252645135, %edi      # t3 = t1 & 0x0F0F0F0F
  subq %rdi, %rax            # 리턴 값 t2 - t3
  ret

z * 48이 곱셈 인스트럭션이 아니라 leaq(×3) + salq $4(×16)의 조합으로 구현된 점을 보세요. 컴파일러는 곱셈을 이렇게 쪼개는 걸 좋아합니다. 그리고 하나의 레지스터(%rax)가 3z → 48z → t4 순으로 여러 프로그램 값을 재사용하며 흘러가는 것도 전형적인 패턴입니다.

관용구 하나: xorq %rcx, %rcx. 자기 자신과 XOR하면 항상 0이므로 이것은 "레지스터를 0으로 만들기"입니다. movq $0, %rcx보다 인코딩 바이트 수가 적어서 컴파일러가 선호합니다.

3.5.5 특수 산술연산 — 128비트 곱셈과 나눗셈

64비트 × 64비트의 완전한 곱은 128비트가 필요합니다. 인텔은 16바이트를 옥트워드(oct word) 라고 부릅니다. x86-64는 %rdx(상위 64비트)와 %rax(하위 64비트)를 한 덩어리 128비트 레지스터처럼 사용해 이를 지원합니다.

Instruction	Effect	설명
imulq S	R[%rdx]:R[%rax] ← S × R[%rax]	부호형 완전 곱셈
mulq S	R[%rdx]:R[%rax] ← S × R[%rax]	비부호형 완전 곱셈
cqto	R[%rdx]:R[%rax] ← SignExtend(R[%rax])	옥트워드로 변환
idivq S	R[%rdx] ← R[%rdx]:R[%rax] mod S
R[%rax] ← R[%rdx]:R[%rax] ÷ S	부호형 나눗셈
divq S	R[%rdx] ← R[%rdx]:R[%rax] mod S
R[%rax] ← R[%rdx]:R[%rax] ÷ S	비부호형 나눗셈

imulq는 이름이 같은 두 인스트럭션입니다. 오퍼랜드 개수로 구분합니다.

2 오퍼랜드 imulq S, D: 64비트 × 64비트 → 64비트(절삭). 비부호형/2의 보수 모두 동일한 비트 동작이므로 하나면 충분.
1 오퍼랜드 imulq S / mulq S: 완전한 128비트 곱. 한 인자는 반드시 %rax에, 결과는 %rdx:%rax에.
void store_uprod(uint128_t *dest, uint64_t x, uint64_t y) { *dest = x * (uint128_t) y; }

  dest in %rdi, x in %rsi, y in %rdx
store_uprod:
  movq %rsi, %rax      # x를 피승수로 복사
  mulq %rdx            # y를 곱함
  movq %rax, (%rdi)    # 하위 8바이트를 dest에 저장
  movq %rdx, 8(%rdi)   # 상위 8바이트를 dest+8에 저장
  ret

리틀 엔디안 머신이므로 상위 바이트가 높은 주소(8(%rdi))로 갑니다.

나눗셈도 단일 오퍼랜드입니다. 피제수(dividend)를 128비트로 %rdx:%rax에 넣고, 제수(divisor)를 오퍼랜드로 줍니다. 결과는 몫이 %rax, 나머지가 %rdx.

문제는 대부분의 경우 피제수가 그냥 64비트 값이라는 점입니다. 이때 %rax에 값을 넣고 %rdx를 채워야 하는데:

비부호형: %rdx를 전부 0으로 (보통 %rdx를 0으로 설정한 뒤 divq)
부호형: %rdx를 %rax의 부호비트로 채움 → cqto 가 이 일을 합니다. 오퍼랜드가 없고, %rax의 부호비트를 암묵적으로 읽어 %rdx 전체에 복사합니다.
void remdiv(long x, long y, long *qp, long *rp) { *qp = x/y; *rp = x%y; }

  x in %rdi, y in %rsi, qp in %rdx, rp in %rcx
remdiv:
  movq %rdx, %r8     # qp를 다른 곳으로 대피 (%rdx가 나눗셈에 필요하므로!)
  movq %rdi, %rax    # x를 피제수 하위 8바이트로
  cqto               # 상위 8바이트로 부호 확장
  idivq %rsi         # y로 나눔
  movq %rax, (%r8)   # 몫을 qp에 저장
  movq %rdx, (%rcx)  # 나머지를 rp에 저장
  ret

💡 인자 qp가 왜 %r8로 옮겨졌는지가 이 예제의 핵심입니다. %rdx는 세 번째 인자 전달 레지스터이면서 동시에 나눗셈이 반드시 사용하는 레지스터입니다. 충돌하므로 대피시킨 것입니다.

참고로 이 인스트럭션은 인텔 문서에서는 cqo라고 부릅니다. ATT 이름과 인텔 이름이 다른 몇 안 되는 사례입니다.

3.6 제어문

지금까지는 인스트럭션이 순서대로 실행되는 직선 코드만 다뤘습니다. 조건부 실행을 구현하는 저수준 방법은 두 가지입니다.

제어 흐름을 바꾼다 — 조건부 점프 (일반적이고 보편적)
데이터 흐름을 바꾼다 — 조건부 이동 (제한적이지만 최신 프로세서에서 빠름)
3.6.1 조건 코드

CPU는 정수 레지스터와 별개로, 가장 최근 산술/논리 연산의 성질을 기록하는 단일 비트 조건 코드 레지스터를 유지합니다.

플래그	이름	의미
CF	캐리 플래그	최상위 비트에서 받아올림 발생. 비부호형 오버플로우 검출
ZF	영 플래그	결과가 0
SF	부호 플래그	결과가 음수
OF	오버플로우 플래그	2의 보수(부호형) 오버플로우 발생

t = a + b를 ADD로 수행했다면 조건 코드는 다음과 같이 설정됩니다.

플래그	설정되는 조건 (C 수식)
CF	(unsigned) t < (unsigned) a — 비부호형 오버플로우
ZF	(t == 0)
SF	(t < 0)
OF	(a<0 == b<0) && (t<0 != a<0) — 부호형 오버플로우

세부 규칙들:

leaq는 조건 코드를 변경하지 않습니다 (주소 계산용이므로). 그림 3.10의 나머지는 모두 변경합니다.
XOR 같은 논리연산은 CF와 OF를 0으로 설정합니다.
쉬프트는 CF를 "마지막으로 밀려나간 비트"로 설정하고, OF는 0으로 설정합니다.
INC / DEC는 OF와 ZF는 설정하지만 CF는 건드리지 않습니다.
조건 코드만 바꾸는 인스트럭션
Instruction	Based on	설명
CMP S1, S2	S2 − S1	비교 (cmpb/w/l/q)
TEST S1, S2	S1 & S2	시험 (testb/w/l/q)
CMP는 SUB와 똑같이 동작하되 목적지를 갱신하지 않고 조건 코드만 바꿉니다. 두 오퍼랜드가 같으면 ZF가 1이 되고, 나머지 플래그로 대소 관계를 판단합니다.
TEST는 AND와 똑같이 동작하되 목적지를 갱신하지 않습니다. 전형적인 용법은 두 가지 — 같은 오퍼랜드를 반복해서 부호를 보거나(testq %rax, %rax로 음수/0/양수 판별), 한쪽을 마스크로 써서 특정 비트를 검사하는 것입니다.

ATT 형식은 오퍼랜드가 역순이라 cmpq %rsi, %rdi는 "a와 b를 비교"(a가 %rdi, b가 %rsi)입니다. 코드를 읽을 때 가장 자주 실수하는 지점입니다.

3.6.2 조건 코드 사용하기 — SET 인스트럭션

조건 코드를 쓰는 방법은 셋입니다. ① 한 바이트에 0/1 기록(SET), ② 다른 위치로 점프(jump), ③ 조건부 데이터 전송(cmov). 먼저 SET입니다.

Instruction	Synonym	Effect (D ←)	조건
sete D	setz	ZF	같음 / 0
setne D	setnz	~ZF	다름 / 0 아님
sets D		SF	음수
setns D		~SF	음수 아님
setg D	setnle	~(SF^OF) & ~ZF	초과 (부호형 >)
setge D	setnl	~(SF^OF)	이상 (부호형 >=)
setl D	setnge	SF^OF	미만 (부호형 <)
setle D	setng	(SF^OF) | ZF	이하 (부호형 <=)
seta D	setnbe	~CF & ~ZF	above (비부호형 >)
setae D	setnb	~CF	above or equal (비부호형 >=)
setb D	setnae	CF	below (비부호형 <)
setbe D	setna	CF | ZF	below or equal (비부호형 <=)

💡 가장 흔한 오해: SET 인스트럭션의 접미사는 오퍼랜드 크기가 아니라 조건을 나타냅니다. setl은 "set long word"가 아니라 "set less", setb는 "set byte"가 아니라 "set below" 입니다.

SET은 하위 단일 바이트에만 0 또는 1을 씁니다. 그래서 32/64비트 결과를 만들려면 나머지 상위 비트를 0으로 지워야 합니다.

int comp(data_t a, data_t b)   /* return a < b */

  a in %rdi, b in %rsi
comp:
  cmpq %rsi, %rdi      # a:b 비교
  setl %al             # %eax의 하위 바이트를 0 또는 1로
  movzbl %al, %eax     # %eax의 나머지 (그리고 %rax의 상위 4바이트까지) 클리어
  ret
왜 부호형 비교는 SF^OF인가

이 유도는 한 번은 직접 따라가 볼 가치가 있습니다. t = a - b라고 합시다.

오버플로우가 없을 때(OF=0): a < b면 t < 0이므로 SF=1, a >= b면 SF=0. 즉 SF가 곧 답.
오버플로우가 있을 때(OF=1): 음의 오버플로우(t가 양수로 뒤집힘)면 실제로는 a < b, 양의 오버플로우(t가 음수로 뒤집힘)면 실제로는 a > b. 즉 SF가 뒤집힌 답.

따라서 "OF가 1이면 SF를 뒤집는다" = SF ^ OF 가 a < b의 정답입니다. 나머지 부호형 비교는 여기에 ZF를 조합해서 만듭니다.

비부호형은 훨씬 단순합니다. a - b에서 a < b이면 CMP가 캐리 플래그를 세팅하므로, CF와 ZF의 조합으로 판단합니다.

기계어는 타입을 모른다: C와 달리 기계어는 값에 자료형을 연관시키지 않습니다. 대부분의 연산에서 비부호형과 2의 보수가 동일한 비트 동작을 갖기 때문에 같은 인스트럭션을 씁니다. 구분이 필요한 곳은 우측 쉬프트, 나눗셈, 완전 곱셈, 그리고 조건 코드의 조합(부호형 vs 비부호형 비교) 뿐입니다.

3.6.3 점프 인스트럭션
Instruction	Synonym	점프 조건	설명
jmp Label		1	직접 점프
jmp *Operand		1	간접 점프
je Label	jz	ZF	같음 / 0
jne Label	jnz	~ZF	다름
js Label		SF	음수
jns Label		~SF	음수 아님
jg Label	jnle	~(SF^OF) & ~ZF	초과 (부호형)
jge Label	jnl	~(SF^OF)	이상 (부호형)
jl Label	jnge	SF^OF	미만 (부호형)
jle Label	jng	(SF^OF) | ZF	이하 (부호형)
ja Label	jnbe	~CF & ~ZF	above (비부호형)
jae Label	jnb	~CF	above or equal
jb Label	jnae	CF	below
jbe Label	jna	CF | ZF	below or equal
직접 점프: 목적지가 인스트럭션에 인코딩됨. 어셈블리에서는 .L1 같은 레이블로 씁니다.
간접 점프: * 다음에 오퍼랜드 식별자. jmp *%rax는 레지스터 값 자체가 목적지, jmp *(%rax)는 %rax가 가리키는 메모리에서 목적지를 읽어옵니다.
조건부 점프는 직접 점프만 가능합니다.

조건들과 이름은 SET 인스트럭션과 완전히 일치합니다. 하나만 외우면 둘 다 아는 셈입니다.

3.6.4 점프 인스트럭션 인코딩

가장 일반적인 방식은 PC 상대(PC-relative) 입니다. 즉, 목적지 주소를 그대로 적는 게 아니라 "점프 인스트럭션 바로 다음 인스트럭션의 주소"와 목적지의 차이를 인코딩합니다. 오프셋은 1, 2, 4바이트로 인코딩될 수 있고, 두 번째 방식으로 4바이트 절대 주소도 가능합니다.

   0:  48 89 f8    mov  %rdi,%rax
   3:  eb 03       jmp  8 <loop+0x8>
   5:  48 d1 f8    sar  %rax
   8:  48 85 c0    test %rax,%rax
   b:  7f f8       jg   5 <loop+0x5>
   d:  f3 c3       repz retq
첫 점프: 인코딩된 오프셋이 0x03. 다음 인스트럭션 주소 0x5 + 0x03 = 0x8. 맞습니다.
둘째 점프: 오프셋이 0xf8, 1바이트 2의 보수로 −8. 다음 인스트럭션 주소 0xd(13) + (−8) = 0x5. 맞습니다.

💡 PC 상대 주소지정에서 기준이 되는 PC 값은 점프 인스트럭션 자신의 주소가 아니라 그 다음 인스트럭션의 주소입니다. 이 관습은 프로세서가 인스트럭션 실행의 첫 단계로 PC를 갱신하던 초기 구현에서 유래했습니다.

링크 후 같은 코드를 역어셈블하면 주소는 0x4004d0 대역으로 재배치되지만 오프셋 값(0x03, 0xf8)은 그대로입니다. PC 상대 인코딩의 두 가지 이득이 여기서 나옵니다 — (1) 점프를 단 2바이트로 간결하게 인코딩할 수 있고, (2) 목적 코드를 수정 없이 메모리의 다른 위치로 옮길 수 있습니다. 이 성질은 7장 링커에서 다시 중요해집니다.

추가정보 — rep; ret는 뭔가요? 함수 끝에 rep; ret(역어셈블하면 repz retq)가 붙는 걸 자주 보게 됩니다. rep는 원래 반복적인 스트링 연산용인데 여기선 전혀 무관해 보입니다. 정답은 AMD의 컴파일러 개발자 가이드라인에 있습니다 — ret 인스트럭션이 조건부 점프의 목적지가 되는 것을 피하라는 권고 때문입니다. AMD 프로세서는 ret가 점프 목적지로 실행되면 리턴 주소를 제대로 예측하지 못합니다. rep를 일종의 no-op으로 끼워 넣으면 코드 동작은 그대로면서 속도만 좋아집니다. 앞으로 나오는 rep/repz는 안심하고 무시해도 됩니다.

3.6.5 조건부 분기를 조건 제어로 구현하기

C의 조건문을 기계어로 옮기는 가장 일반적인 방법입니다. 책은 "goto 코드" 라는 중간 표현을 써서 설명합니다 — 어셈블리의 제어 흐름을 그대로 흉내 낸 C 코드입니다. (실제 프로그래밍에서 goto는 나쁜 스타일이지만, 여기서는 설명 도구입니다.)

long absdiff_se(long x, long y) {      long gotodiff_se(long x, long y) {
    long result;                           long result;
    if (x < y) {                           if (x >= y)
        lt_cnt++;                              goto x_ge_y;
        result = y - x;                    lt_cnt++;
    } else {                               result = y - x;
        ge_cnt++;                          return result;
        result = x - y;                x_ge_y:
    }                                      ge_cnt++;
    return result;                         result = x - y;
}                                          return result;
                                       }
  x in %rdi, y in %rsi
absdiff_se:
  cmpq %rsi, %rdi            # x:y 비교
  jge .L2                    # >= 면 x_ge_y로
  addq $1, lt_cnt(%rip)      # lt_cnt++
  movq %rsi, %rax
  subq %rdi, %rax            # result = y - x
  ret
.L2:                         # x_ge_y:
  addq $1, ge_cnt(%rip)      # ge_cnt++
  movq %rdi, %rax
  subq %rsi, %rax            # result = x - y
  ret

일반적인 if-else의 번역 템플릿:

t = test-expr;
if (!t)
    goto false;
  then-statement
  goto done;
false:
  else-statement
done:

즉 컴파일러는 then-문과 else-문에 대해 별도의 코드 블록을 만들고, 정확히 하나만 실행되도록 조건부 + 무조건 분기를 삽입합니다.

추가정보 — 읽는 순서: 책의 그림들은 (a) 원본 C → (c) 어셈블리 → (b) goto 버전 순으로 만들어졌지만, 읽을 때는 (a) → (b) → (c) 순서를 권장합니다. 머신 코드를 C로 해석한 중간 단계를 먼저 보면 실제 어셈블리가 훨씬 쉽게 들어옵니다.

3.6.6 조건부 이동으로 조건부 분기 구현하기

조건부 점프는 단순하지만 최신 프로세서에서 매우 비효율적일 수 있습니다. 대안은 양쪽 결과를 모두 계산한 뒤 조건에 따라 하나만 고르는 것입니다.

long absdiff(long x, long y) {      long cmovdiff(long x, long y) {
    long result;                        long rval = y-x;
    if (x < y)                          long eval = x-y;
        result = y - x;                 long ntest = x >= y;
    else                                /* 아래 줄은 단일 인스트럭션 */
        result = x - y;                 if (ntest) rval = eval;
    return result;                      return rval;
}                                   }
  x in %rdi, y in %rsi
absdiff:
  movq %rsi, %rax
  subq %rdi, %rax     # rval = y-x
  movq %rdi, %rdx
  subq %rsi, %rdx     # eval = x-y
  cmpq %rsi, %rdi     # x:y 비교
  cmovge %rdx, %rax   # >= 이면 rval = eval
  ret
왜 이게 더 빠른가 — 파이프라인과 분기 예측

프로세서는 인스트럭션을 여러 단계(인출 → 해독 → 메모리 읽기 → 연산 → 쓰기 → PC 갱신)로 쪼개 파이프라인으로 겹쳐 처리합니다. 파이프라인을 채우려면 앞으로 실행할 인스트럭션 순서를 한참 미리 알아야 합니다.

그런데 조건부 점프를 만나면, 분기 조건 계산이 끝나기 전까지는 어느 쪽으로 갈지 알 수 없습니다. 그래서 프로세서는 분기 예측 회로로 추측합니다(최신 설계는 90%대 적중률을 노립니다). 예측이 맞으면 파이프라인은 가득 찬 채로 돌지만, 틀리면 이미 해둔 작업을 전부 버리고 올바른 위치에서 다시 채워야 합니다. 이 예측 오류 손실이 대략 15~30 클럭 사이클입니다.

인텔 Haswell에서 absdiff를 실제로 측정한 결과:

구현 방식	분기 예측이 쉬울 때	분기 패턴이 랜덤일 때
조건부 점프	약 8 사이클	약 17.5 사이클
조건부 이동	약 8 사이클	약 8 사이클

여기서 예측 오류 손실을 역산하면 약 19 사이클이 나옵니다. 즉 함수 실행 시간이 8~27 사이클 사이에서 출렁입니다. 반면 조건부 이동 코드는 제어 흐름이 데이터와 무관하므로 데이터에 상관없이 일정하게 8 사이클입니다.

추가정보 — 19 사이클은 어떻게 계산했나 예측 오류 확률을 p, 오류 없을 때 실행 시간을 T_OK, 오류 손실을 T_MP라 하면 T_avg(p) = (1−p)·T_OK + p·(T_OK + T_MP) = T_OK + p·T_MP. 랜덤 패턴은 p = 0.5이므로 T_ran = T_OK + 0.5·T_MP, 따라서 T_MP = 2(T_ran − T_OK). T_OK = 8, T_ran = 17.5를 넣으면 T_MP = 2 × 9.5 = 19.

그림 3.18 — 조건부 이동 인스트럭션
Instruction	Synonym	이동 조건	설명
cmove S,R	cmovz	ZF	같음 / 0
cmovne S,R	cmovnz	~ZF	다름
cmovs S,R		SF	음수
cmovns S,R		~SF	음수 아님
cmovg S,R	cmovnle	~(SF^OF) & ~ZF	초과 (부호형)
cmovge S,R	cmovnl	~(SF^OF)	이상 (부호형)
cmovl S,R	cmovnge	SF^OF	미만 (부호형)
cmovle S,R	cmovng	(SF^OF) | ZF	이하 (부호형)
cmova S,R	cmovnbe	~CF & ~ZF	above (비부호형)
cmovae S,R	cmovnb	~CF	above or equal
cmovb S,R	cmovnae	CF	below
cmovbe S,R	cmovna	CF | ZF	below or equal

특징 두 가지:

소스/목적지는 16, 32, 64비트. 단일 바이트 조건부 이동은 지원되지 않습니다.
movw/movl처럼 이름에 크기를 인코딩하지 않습니다. 어셈블러가 목적지 레지스터 이름으로 오퍼랜드 길이를 추론하므로 모든 길이에 같은 이름을 씁니다.
조건부 이동을 쓰면 안 되는 경우

일반적인 조건부 수식 v = test-expr ? then-expr : else-expr;를 조건부 이동으로 옮기면 이렇게 됩니다.

vt = then-expr;
ve = else-expr;
t  = test-expr;
if (!t) v = ve;

핵심은 테스트 결과와 무관하게 then-expr와 else-expr가 둘 다 실행된다는 점입니다. 여기서 두 가지 문제가 생깁니다.

(1) 부수효과나 오류가 있으면 안 됩니다.

long cread(long *xp) { return (xp ? *xp : 0); }

  ★ 잘못된 구현 ★
cread:
  movq (%rdi), %rax   # v = *xp   ← xp가 NULL이어도 역참조해버림!
  testq %rdi, %rdi
  movl $0, %edx
  cmove %rdx, %rax
  ret

테스트가 실패해도 *xp 역참조가 일어나 널 포인터 오류가 납니다. 이 코드는 반드시 분기로 컴파일되어야 합니다. (앞선 absdiff_se 예제에서 전역 카운터를 증가시키는 부수효과를 일부러 넣은 것도, GCC가 조건부 이동을 못 쓰게 만들기 위한 장치였습니다.)

(2) 계산량이 많으면 손해입니다. 한쪽 계산이 무거운데 조건이 맞지 않으면 그 노력은 통째로 낭비됩니다. 컴파일러는 "낭비되는 계산량" vs "분기 예측 오류 손실"을 저울질해야 하는데, 분기가 얼마나 예측 가능한지 미리 알 수 없어서 정확한 판단이 어렵습니다. 실험해 보면 GCC는 양쪽 수식이 add 하나 수준으로 아주 간단할 때만 조건부 이동을 쓰고, 예측 오류 비용이 복잡한 계산 비용보다 큰 경우에도 조건부 제어 이동 쪽을 택하는 경향이 있습니다.

3.6.7 반복문

기계어에는 반복문 인스트럭션이 없습니다. 조건부 테스트와 점프의 조합으로 만듭니다. GCC는 두 가지 기본 패턴을 씁니다.

do-while — 가장 단순한 형태
do body-statement while (test-expr);

↓

loop:
  body-statement
  t = test-expr;
  if (t) goto loop;
long fact_do(long n) {
    long result = 1;
    do { result *= n; n = n-1; } while (n > 1);
    return result;
}

  n in %rdi
fact_do:
  movl $1, %eax       # result = 1
.L2:                  # loop:
  imulq %rdi, %rax    # result *= n
  subq $1, %rdi       # n--
  cmpq $1, %rdi       # n:1 비교
  jg .L2              # > 이면 loop로
  rep; ret

추가정보 — 반복문 역엔지니어링 전략 생성된 어셈블리를 원본 C에 대응시키는 열쇠는 프로그램 값과 레지스터의 매핑을 찾는 것입니다. 방법은 이렇습니다 — 반복문 이전에 어떤 레지스터가 초기화되는지, 루프 안에서 어떻게 갱신되는지, 무엇이 테스트되는지, 루프 이후에 무엇이 쓰이는지를 차례로 살펴보세요. 위 예제에서 %rax가 1로 초기화되고(movl $1, %eax는 %rax 상위 4바이트까지 0으로 만든다는 점을 기억!) 곱셈으로 갱신되며 리턴에 쓰이므로, %rax = result라고 결론 내릴 수 있습니다. 다만 실전에서는 훨씬 어렵습니다. 컴파일러는 계산 순서를 바꾸고, C의 일부 변수는 기계어에 흔적이 없으며, 소스에 없던 새 값이 등장하기도 하고, 여러 프로그램 값이 한 레지스터에 매핑되기도 합니다.

while — 번역 방법이 두 가지

① 중간으로-점프(jump to middle) — GCC -Og에서 사용

  goto test;
loop:
  body-statement
test:
  t = test-expr;
  if (t) goto loop;
  n in %rdi
fact_while:
  movl $1, %eax     # result = 1
  jmp .L5           # test로 점프
.L6:                # loop:
  imulq %rdi, %rax
  subq $1, %rdi
.L5:                # test:
  cmpq $1, %rdi
  jg .L6
  rep; ret

② 조건형-do(guarded do) — GCC -O1 이상에서 사용. 초기 테스트가 실패하면 루프 전체를 건너뛰는 조건부 분기를 앞에 두고, 나머지는 do-while로 만듭니다.

  t = test-expr;
  if (!t) goto done;
loop:
  body-statement
  t = test-expr;
  if (t) goto loop;
done:
  n in %rdi
fact_while:
  cmpq $1, %rdi     # n:1
  jle .L7           # <= 이면 done
  movl $1, %eax     # result = 1
.L6:                # loop:
  imulq %rdi, %rax
  subq $1, %rdi
  cmpq $1, %rdi
  jne .L6           # != 이면 loop  ← 주목!
  rep; ret
.L7:                # done:
  movl $1, %eax
  ret

💡 여기서 흥미로운 최적화가 보입니다. 원본 C의 테스트는 n > 1인데 루프 안의 테스트는 n != 1 로 바뀌었습니다. 컴파일러는 "이 루프는 n > 1일 때만 진입하고, n은 1씩 줄어드니 1보다 작아지기 전에 반드시 1을 거친다"고 추론한 것입니다. 조건형-do 방식이 이런 최적화를 가능하게 해줍니다.

for — while로 환원
for (init-expr; test-expr; update-expr) body-statement

는 (3.29 연습문제의 continue 예외를 빼면) 다음과 동일합니다.

init-expr;
while (test-expr) { body-statement; update-expr; }

그다음은 while의 두 전략 중 하나를 그대로 따릅니다. 중간으로-점프 버전:

init-expr;
goto test;
loop:
  body-statement
  update-expr;
test:
  t = test-expr;
  if (t) goto loop;
long fact_for(long n) {
    long i; long result = 1;
    for (i = 2; i <= n; i++) result *= i;
    return result;
}

  n in %rdi
fact_for:
  movl $1, %eax     # result = 1
  movl $2, %edx     # i = 2
  jmp .L8           # test로
.L9:                # loop:
  imulq %rdx, %rax  # result *= i
  addq $1, %rdx     # i++
.L8:                # test:
  cmpq %rdi, %rdx   # i:n
  jle .L9
  rep; ret

결론: C의 세 반복문(do-while, while, for)은 모두 하나 이상의 조건부 분기를 갖는 단순한 전략으로 번역됩니다. 조건부 제어 전환이 반복문 번역의 기본 메커니즘입니다.

3.6.8 switch문

switch는 정수 인덱스에 따른 다중 분기입니다. case가 많을 때 유용하고, 점프 테이블이라는 자료구조로 효율적으로 구현됩니다.

점프 테이블은 원소 i가 "인덱스가 i일 때 실행할 코드 블록의 주소"인 배열입니다. 장점이 결정적입니다 — case 개수와 무관하게 실행 시간이 일정합니다. 수백 개의 case가 있어도 점프 테이블 접근 한 번이면 됩니다. if-else 연쇄는 case 수에 비례해 느려지는 것과 대비됩니다.

GCC는 case 값의 밀집도와 개수를 보고 번역 방법을 고릅니다. case가 4개 이상이고 값들이 좁은 범위에 분포하면 점프 테이블을 씁니다.

void switch_eg(long x, long n, long *dest) {
    long val = x;
    switch (n) {
    case 100: val *= 13;        break;
    case 102: val += 10;        /* fall through */
    case 103: val += 11;        break;
    case 104:
    case 106: val *= val;       break;
    default:  val = 0;
    }
    *dest = val;
}

이 예제는 까다로운 특징을 한꺼번에 담고 있습니다 — 값이 연속적이지 않고(101, 105 없음), 한 블록에 레이블이 여럿이며(104, 106), break 없이 다음 case로 흘러가는 경우(102 → 103)가 있습니다.

  x in %rdi, n in %rsi, dest in %rdx
switch_eg:
  subq $100, %rsi         # index = n - 100
  cmpq $6, %rsi           # index:6
  ja .L8                  # 초과면 loc_def로  ← 비부호형 비교!
  jmp *.L4(,%rsi,8)       # goto *jt[index]
.L3:                      # loc_A: (case 100)
  leaq (%rdi,%rdi,2), %rax   # 3*x
  leaq (%rdi,%rax,4), %rdi   # val = 13*x
  jmp .L2
.L5:                      # loc_B: (case 102)
  addq $10, %rdi
                          #  ← jmp 없음! 다음 블록으로 흘러감 (fall through)
.L6:                      # loc_C: (case 103)
  addq $11, %rdi
  jmp .L2
.L7:                      # loc_D: (case 104, 106)
  imulq %rdi, %rdi
  jmp .L2
.L8:                      # loc_def:
  movl $0, %edi
.L2:                      # done:
  movq %rdi, (%rdx)
  ret

점프 테이블 자체는 읽기 전용 데이터 세그먼트에 놓입니다.

  .section .rodata
  .align 8              # 주소를 8의 배수로 정렬
.L4:
  .quad .L3             # case 100: loc_A
  .quad .L8             # case 101: loc_def
  .quad .L5             # case 102: loc_B
  .quad .L6             # case 103: loc_C
  .quad .L7             # case 104: loc_D
  .quad .L8             # case 105: loc_def
  .quad .L7             # case 106: loc_D

여기서 배울 세 가지 기법:

인덱스 정규화: case 값이 100~106이므로 n - 100을 계산해 0~6 범위로 옮깁니다.
비부호형 트릭: index를 비부호형으로 취급하면, 음수는 2의 보수 특성상 아주 큰 양수가 됩니다. 따라서 "0 이상이고 6 이하인가"라는 두 번의 검사가 ja(unsigned >) 한 번으로 끝납니다. 범위 검사를 반으로 줄이는 매우 흔한 관용구입니다.
테이블에 중복 항목 허용: 다중 레이블(104, 106)은 같은 레이블(.L7)을 두 번 넣어서, 빠진 case(101, 105)는 default 레이블(.L8)을 넣어서 처리합니다. fall-through는 코드 블록 끝의 jmp를 생략하는 것만으로 자연스럽게 구현됩니다.

jmp *.L4(,%rsi,8)의 *는 간접 점프를 의미하고, %rsi(=index)를 8바이트 단위로 스케일링해 테이블을 배열처럼 참조합니다. (배열 참조가 기계어로 번역되는 방식은 3편 3.8절에서 다룹니다.)

3.7 프로시저

프로시저 호출은 소프트웨어의 주요 추상화입니다. "무엇을 계산하는가"라는 인터페이스만 노출하고 구현은 감춥니다. 언어마다 함수, 메소드, 서브루틴, 핸들러 등으로 불리지만 공통 특징을 공유합니다.

프로시저 P가 Q를 호출하고 다시 P로 돌아온다고 할 때, 기계 수준에서 처리해야 할 일은 셋입니다.

메커니즘	해야 할 일
제어권 전달	진입 시 PC를 Q의 시작 주소로, 리턴 시 P의 호출 다음 인스트럭션으로
데이터 전달	P → Q로 매개변수, Q → P로 리턴 값
메모리 할당과 반납	Q가 시작할 때 지역변수 공간 할당, 리턴할 때 반납

핵심 설계 철학은 최소주의입니다. 호출 오버헤드를 줄이기 위해, 각 프로시저는 자기가 실제로 필요로 하는 메커니즘만 구현합니다.

3.7.1 런타임 스택

프로시저 호출은 후입선출(LIFO) 패턴을 따릅니다. P가 Q를 호출하면 Q가 실행되는 동안 P는 정지하고, Q가 끝나면 Q의 저장공간은 반납됩니다. 이 패턴이 스택 자료구조와 정확히 맞아떨어집니다.

x86-64 스택은 작은 주소 방향으로 성장하고, %rsp가 스택 최상위를 가리킵니다. 초기화가 필요 없는 공간은 %rsp를 감소시키는 것만으로 할당되고, 증가시키면 반납됩니다.

레지스터에 담을 수 있는 것보다 많은 저장공간이 필요할 때 프로시저는 스택에 스택 프레임을 할당합니다.

                    Stack "bottom"
        ┌────────────────────────────┐   높은 주소
        │      Earlier frames        │
        ├────────────────────────────┤
        │      Argument n            │
        │          ...               │   ← P의 프레임
        │      Argument 7            │
        ├────────────────────────────┤
        │      Return address        │   ← call이 푸시 (P의 프레임에 속함)
        ├────────────────────────────┤
        │      Saved registers       │
        │      Local variables       │   ← Q의 프레임
        │      Argument build area   │
        └────────────────────────────┘ ← %rsp  낮은 주소
                    Stack "top"

몇 가지 중요한 점:

현재 실행 중인 프로시저의 프레임이 항상 스택 맨 위에 있습니다.
리턴 주소는 P의 스택 프레임에 속하는 것으로 간주합니다 (P에 관계된 상태를 저장하므로).
대부분의 프레임은 프로시저 시작 시 할당되는 고정 크기입니다. 가변 크기는 3.10.5절에서 다룹니다.
정수/포인터 인자는 최대 6개까지 레지스터로 전달되고, 더 필요하면 호출 전에 호출자의 프레임에 저장합니다.

💡 많은 함수는 스택 프레임 자체를 요구하지 않습니다. 모든 지역변수를 레지스터에 담을 수 있고 다른 함수를 하나도 호출하지 않을 때 그렇습니다(호출 트리의 끝에 있다고 해서 나뭇잎 프로시저라고 부릅니다). 지금까지 본 함수들이 전부 그런 경우였습니다.

3.7.2 제어의 이동
Instruction	설명
call Label	프로시저 호출 (직접)
call *Operand	프로시저 호출 (간접)
ret	호출에서 리턴
call Q: 리턴 주소 A를 스택에 푸시하고 PC를 Q의 시작으로 설정. 여기서 A는 call 인스트럭션 바로 다음 인스트럭션의 주소입니다.
ret: 스택에서 A를 팝하고 PC를 A로 설정.

(역어셈블 출력에서는 callq, retq로 표기되는데, 접미사 q는 IA32가 아닌 x86-64 버전임을 강조할 뿐입니다. 어셈블리에서는 양쪽 다 쓸 수 있습니다.)

아래는 main → top(100) → leaf(95)로 이어지는 호출의 실제 추적입니다. 각 행은 인스트럭션 실행 직전의 상태입니다.

leaf:                                 top:
  400540: lea 0x2(%rdi),%rax  L1        400545: sub $0x5,%rdi       T1
  400544: retq                L2        400549: callq 400540 <leaf> T2
                                        40054e: add %rax,%rax       T3
main:                                   400551: retq                T4
  40055b: callq 400545 <top>  M1
  400560: mov %rax,%rdx       M2
Label	PC	Instruction	%rdi	%rax	%rsp	*%rsp	설명
M1	0x40055b	callq	100	—	0x7fffffffe820	—	top(100) 호출
T1	0x400545	sub	100	—	0x7fffffffe818	0x400560	top 진입
T2	0x400549	callq	95	—	0x7fffffffe818	0x400560	leaf(95) 호출
L1	0x400540	lea	95	—	0x7fffffffe810	0x40054e	leaf 진입
L2	0x400544	retq	—	97	0x7fffffffe810	0x40054e	leaf가 97 리턴
T3	0x40054e	add	—	97	0x7fffffffe818	0x400560	top 재개
T4	0x400551	retq	—	194	0x7fffffffe818	0x400560	top이 194 리턴
M2	0x400560	mov	—	194	0x7fffffffe820	—	main 재개

%rsp가 호출마다 8씩 감소했다가(리턴 주소 푸시) 리턴할 때마다 원래 값으로 복원되는 것, 그리고 *%rsp(스택 탑에 있는 값)가 정확히 해당 호출의 리턴 주소라는 것을 확인하세요. C의 표준 호출/리턴 방식이 스택의 LIFO 관리와 얼마나 자연스럽게 맞물리는지 보여주는 표입니다.

3.7.3 데이터 전송

x86-64에서는 최대 6개의 정수형(정수·포인터) 인자가 레지스터로 전달됩니다. 인자 순서에 따라 정해진 레지스터가 쓰이고, 데이터 크기에 따라 이름이 달라집니다.

오퍼랜드 크기	인자 1	인자 2	인자 3	인자 4	인자 5	인자 6
64비트	%rdi	%rsi	%rdx	%rcx	%r8	%r9
32비트	%edi	%esi	%edx	%ecx	%r8d	%r9d
16비트	%di	%si	%dx	%cx	%r8w	%r9w
8비트	%dil	%sil	%dl	%cl	%r8b	%r9b

리턴 값은 %rax 로 돌아옵니다.

인자가 7개 이상이면 나머지는 스택으로 전달됩니다. 호출자 P가 인자 7~n을 위한 공간을 자기 프레임에 할당하고, 인자 7을 스택 탑에 놓는 식으로 저장합니다. 이때 모든 데이터 크기는 8의 배수로 반올림됩니다.

void proc(long a1, long *a1p, int a2, int *a2p,
          short a3, short *a3p, char a4, char *a4p) {
    *a1p += a1;  *a2p += a2;  *a3p += a3;  *a4p += a4;
}

  a1 in %rdi(64), a1p in %rsi(64), a2 in %edx(32), a2p in %rcx(64),
  a3 in %r8w(16), a3p in %r9(64), a4 at %rsp+8(8), a4p at %rsp+16(64)
proc:
  movq 16(%rsp), %rax    # a4p 가져오기 (64비트)
  addq %rdi, (%rsi)      # *a1p += a1  (64비트)
  addl %edx, (%rcx)      # *a2p += a2  (32비트)
  addw %r8w, (%r9)       # *a3p += a3  (16비트)
  movl 8(%rsp), %edx     # a4 가져오기 (8비트)
  addb %dl, (%rax)       # *a4p += a4  (8비트)
  ret
오퍼랜드 크기마다 다른 ADD 변형이 쓰입니다: addq(long) / addl(int) / addw(short) / addb(char).
스택 인자가 %rsp+8, %rsp+16에 있는 이유는 리턴 주소가 %rsp(오프셋 0)에 푸시되어 있기 때문입니다.
movl 8(%rsp), %edx가 4바이트를 읽지만 이어지는 addb는 하위 1바이트만 쓰는 점도 눈여겨볼 만합니다.
3.7.4 스택에서의 지역 저장공간

지역 데이터가 반드시 메모리에 있어야 하는 경우가 있습니다.

지역 데이터를 전부 담기엔 레지스터가 부족할 때
지역변수에 & 연산자가 사용되어 주소를 만들 수 있어야 할 때
지역변수가 배열이나 구조체여서 배열/구조체 참조로 접근해야 할 때

이럴 때 프로시저는 %rsp를 감소시켜 스택 프레임에 "Local variables" 영역을 만듭니다.

long caller() {
    long arg1 = 534;
    long arg2 = 1057;
    long sum = swap_add(&arg1, &arg2);
    long diff = arg1 - arg2;
    return sum * diff;
}

caller:
  subq $16, %rsp          # 스택 프레임 16바이트 할당
  movq $534, (%rsp)       # arg1에 534 저장
  movq $1057, 8(%rsp)     # arg2에 1057 저장
  leaq 8(%rsp), %rsi      # &arg2를 두 번째 인자로
  movq %rsp, %rdi         # &arg1을 첫 번째 인자로
  call swap_add
  movq (%rsp), %rdx       # arg1 읽기 (swap_add가 바꿔놓았음)
  subq 8(%rsp), %rdx      # diff = arg1 - arg2
  imulq %rdx, %rax        # sum * diff
  addq $16, %rsp          # 스택 프레임 반납
  ret

&arg1, &arg2를 만들어야 하므로 두 지역변수는 레지스터가 아니라 스택에 놓였습니다(오프셋 0과 8). leaq가 주소 계산에 쓰이는 원래 용도로 등장하는 것도 확인하세요.

좀 더 복잡한 예로, 앞의 proc(인자 8개)를 호출하는 함수를 봅시다.

long call_proc() {
    long x1 = 1; int x2 = 2; short x3 = 3; char x4 = 4;
    proc(x1, &x1, x2, &x2, x3, &x3, x4, &x4);
    return (x1+x2) * (x3-x4);
}

call_proc:
  subq $32, %rsp          # 32바이트 프레임 할당
  movq $1, 24(%rsp)       # x1
  movl $2, 20(%rsp)       # x2
  movw $3, 18(%rsp)       # x3
  movb $4, 17(%rsp)       # x4
  leaq 17(%rsp), %rax
  movq %rax, 8(%rsp)      # &x4를 인자 8로 (스택)
  movl $4, (%rsp)         # 4를 인자 7로 (스택)
  leaq 18(%rsp), %r9      # &x3를 인자 6으로
  movl $3, %r8d           # 3을 인자 5로
  leaq 20(%rsp), %rcx     # &x2를 인자 4로
  movl $2, %edx           # 2를 인자 3으로
  leaq 24(%rsp), %rsi     # &x1을 인자 2로
  movl $1, %edi           # 1을 인자 1로
  call proc
  movslq 20(%rsp), %rdx   # x2를 long으로 변환
  addq 24(%rsp), %rdx     # x1+x2
  movswl 18(%rsp), %eax   # x3를 int로
  movsbl 17(%rsp), %ecx   # x4를 int로
  subl %ecx, %eax         # x3-x4
  cltq                    # long으로 변환
  imulq %rdx, %rax
  addq $32, %rsp          # 프레임 반납
  ret

스택 프레임 배치(%rsp 기준 오프셋):

오프셋	내용
24~31	x1 (long, 8바이트)
20~23	x2 (int, 4바이트)
18~19	x3 (short, 2바이트)
17	x4 (char, 1바이트)
8~15	인자 8 (&x4)
0~7	인자 7 (값 4)

코드의 절반 이상(14줄 중 대부분)이 호출 준비에 쓰인다는 점을 보세요. 그리고 proc 안에서는 인자 7·8이 %rsp+8, %rsp+16에 보이는데, 여기(call_proc)서는 %rsp+0, %rsp+8입니다. 차이 8바이트가 바로 call이 푸시한 리턴 주소입니다.

3.7.5 레지스터를 이용하는 지역 저장소

레지스터는 모든 프로시저가 공유하는 단일 자원입니다. 피호출자가 호출자의 값을 덮어쓰면 안 되므로, x86-64는 라이브러리를 포함한 모든 프로시저가 지켜야 할 레지스터 사용 관습을 정해두었습니다.

분류	레지스터	책임
피호출자-저장 (callee-saved)	%rbx, %rbp, %r12~%r15	Q(피호출자) 가 값을 보존해야 함. 안 쓰거나, 쓰려면 먼저 스택에 푸시하고 리턴 전에 팝
호출자-저장 (caller-saved)	%rsp를 제외한 나머지 전부	P(호출자) 가 필요하면 호출 전에 직접 저장. Q는 자유롭게 변경 가능

두 이름 모두 "누가 저장할 책임이 있는가" 라는 관점에서 붙은 것입니다.

long P(long x, long y) { long u = Q(y); long v = Q(x); return u + v; }

  x in %rdi, y in %rsi
P:
  pushq %rbp        # %rbp 저장
  pushq %rbx        # %rbx 저장
  subq $8, %rsp     # 스택 프레임 정렬
  movq %rdi, %rbp   # x를 피호출자-저장 레지스터에 보관
  movq %rsi, %rdi   # y를 첫 번째 인자로
  call Q            # Q(y)
  movq %rax, %rbx   # 결과를 피호출자-저장 레지스터에 보관
  movq %rbp, %rdi   # x를 첫 번째 인자로
  call Q            # Q(x)
  addq %rbx, %rax   # Q(y) + Q(x)
  addq $8, %rsp
  popq %rbx         # %rbx 복원
  popq %rbp         # %rbp 복원
  ret

P는 두 값(x, 그리고 Q(y)의 결과)을 호출을 건너 살려둬야 합니다. 그래서 피호출자-저장 레지스터 %rbp와 %rbx를 쓰되, 먼저 원래 값을 스택에 푸시하고 마지막에 복원합니다. 푸시한 순서와 정확히 역순으로 팝되는 것이 스택 LIFO의 직접적 표현입니다.

3.7.6 재귀 프로시저

앞의 관습들만으로 재귀가 자동으로 동작합니다. 각 호출은 스택에 자신만의 사적인 공간을 가지므로 서로 다른 호출의 지역변수가 간섭하지 않고, 할당/반납 시점이 호출/리턴 순서와 자연스럽게 일치합니다.

long rfact(long n) {
    long result;
    if (n <= 1) result = 1;
    else result = n * rfact(n-1);
    return result;
}

  n in %rdi
rfact:
  pushq %rbx           # %rbx 저장
  movq %rdi, %rbx      # n을 피호출자-저장 레지스터에 보관
  movl $1, %eax        # 리턴 값 = 1
  cmpq $1, %rdi
  jle .L35             # n <= 1 이면 done
  leaq -1(%rdi), %rdi  # n-1
  call rfact           # rfact(n-1)
  imulq %rbx, %rax     # 결과 × n
.L35:                  # done:
  popq %rbx            # %rbx 복원
  ret

rfact(n-1)이 리턴한 시점에 두 가지가 보장됩니다 — (1) 호출 결과가 %rax에 있고, (2) 인자 n이 %rbx에 그대로 보존되어 있다는 것. (2)는 재귀 호출된 rfact가 피호출자-저장 관습을 지켜 %rbx를 푸시/팝했기 때문입니다.

💡 재귀 함수를 위한 특별한 장치는 아무것도 없습니다. 표준 프로시저 처리 방식(스택 + 레지스터 저장 관습)만으로 충분하며, P가 Q를 부르고 Q가 다시 P를 부르는 상호 재귀 같은 복잡한 형태에서도 똑같이 동작합니다.

