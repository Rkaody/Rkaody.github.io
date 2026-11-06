---
layout: post
title: "pwntools 기초 — 연결, 송수신, 패킹"
date: 2026-11-06
categories: [pwn]
tags: [python, pwntools, dreamhack]
excerpt: "pwntools의 연결, 송수신, 패킹/언패킹 함수 정리와 DreamHack 실습 풀이."
---

pwntools는 CTF 및 시스템 해킹에서 자주 사용되는 Python 익스플로잇 라이브러리다.
로컬 바이너리 실행부터 원격 서버 통신, 바이너리 분석까지 다양한 기능을 제공한다.

## 연결 함수

연결에 성공하면 `pwnlib.tubes` 클래스 객체를 반환한다. 이 객체를 통해 이후의 송수신이 이루어진다.

```python
from pwn import *

p = process("./binary")               # 로컬 바이너리 실행
r = remote("host.example.com", 1234)  # 원격 TCP 연결
s = ssh("user", "host", password="pw")  # SSH 연결
```

`remote()`는 기본적으로 TCP를 사용하고, `typ="udp"` 옵션으로 UDP 연결도 가능하다.

## 데이터 송수신

### 수신 (recv)

| 함수 | 동작 |
|------|------|
| `recv(n)` | 최대 n 바이트 수신 |
| `recvn(n)` | 정확히 n 바이트 수신 |
| `recvline()` | 개행 문자까지 한 줄 수신 |
| `recvuntil(b"str", drop=True)` | 지정 문자열까지 수신. `drop=True`면 해당 문자열 제외 |
| `recvall()` | 연결이 끊길 때까지 전부 수신 |

### 송신 (send)

| 함수 | 동작 |
|------|------|
| `send(data)` | 데이터 전송 |
| `sendline(data)` | 데이터 + `\n` 전송 |
| `sendafter(b"str", data)` | 지정 문자열 수신 후 데이터 전송 |
| `sendlineafter(b"str", data)` | 지정 문자열 수신 후 데이터 + `\n` 전송 |

`sendafter`/`sendlineafter`는 서버의 프롬프트를 기다렸다가 전송하므로, 타이밍이 중요한 상황에서 유용하다.

### 인터랙티브 모드

```python
r.interactive()
```

셸을 획득했거나 플래그를 직접 받아야 할 때 사용한다. 사용자가 직접 입력/출력을 주고받을 수 있다.
`interactive()`를 호출한 경우 `close()`는 별도로 호출하지 않아도 된다.

## 기타 기능

### 로그 레벨

```python
context.log_level = 'debug'  # critical / error / warn / info / debug
```

`debug`로 설정하면 송수신되는 모든 데이터를 출력해준다. 익스플로잇 디버깅 시 유용하다.

### 아키텍처 설정

```python
context.arch = 'amd64'  # i386 / amd64 / arm / aarch64
context.bits = 64
context.os   = 'linux'
```

익스플로잇 대상 바이너리의 아키텍처와 OS를 지정한다. `shellcraft`로 셸코드를 생성하거나 `asm()`으로 어셈블할 때 이 설정을 기준으로 동작한다. 64비트 리눅스 바이너리를 다룰 때는 `context.arch = 'amd64'`를 먼저 설정하는 것이 관행이다.

`context(arch='amd64', os='linux')` 형태로 한 줄에 쓸 수도 있다.

### 패킹 / 언패킹

메모리 주소나 정수 값을 바이트로 변환하거나, 반대로 바이트 데이터를 정수로 복원할 때 사용한다.
pwntools는 기본적으로 **리틀 엔디언** (낮은 주소에 낮은 바이트 저장) 방식으로 변환한다.

```python
p32(0xdeadbeef)   # 정수 → 4바이트
p64(0xdeadbeef)   # 정수 → 8바이트
u32(b"\xef\xbe\xad\xde")  # 4바이트 → 정수
u64(b"\xef\xbe\xad\xde\x00\x00\x00\x00")  # 8바이트 → 정수
```

스택 오버플로우 익스플로잇에서 리턴 주소를 덮어쓸 때 `p64()`가 필수적으로 사용된다.

### GDB 연동

```python
p = process("./binary")
gdb.attach(p)
```

`process()`로 실행한 프로세스에 GDB를 붙여서 디버깅할 수 있다.

### ELF 분석

```python
e = ELF("./binary")
print(e.symbols["main"])   # 함수/변수 주소 조회
print(e.got["puts"])       # GOT 주소 조회
print(e.plt["puts"])       # PLT 주소 조회
```

ELF(Executable and Linkable Format)는 Linux에서 사용하는 실행 파일 형식이다.
pwntools의 `ELF` 클래스로 바이너리의 심볼 테이블, GOT, PLT 등을 분석할 수 있다.

---

## 실습 1 — 랜덤 덧셈 50회

```c
// gcc chall.c -o chall -fno-stack-protector
for (int i = 0; i < 50; i++) {
    alarm(1);  // 1초 제한
    num1 = rand() % 10000;
    num2 = rand() % 10000;
    printf("%d+%d=?\n", num1, num2);
    scanf("%d", &inpt);

    if (inpt != num1 + num2) {
        printf("Wrong...\n");
        return 0;
    }
}
puts(flag);
```

매 라운드마다 `alarm(1)`으로 1초 제한을 건다. 사람이 직접 풀 수 없으므로 pwntools로 자동화해야 한다.

**풀이**

```python
from pwn import *

r = remote('host3.dreamhack.games', 17166)

for i in range(50):
    first_number = int(r.recvuntil(b'+', drop=True))
    second_number = int(r.recvuntil(b'=?\n', drop=True))
    r.sendline(str(first_number + second_number).encode())

r.interactive()
```

`recvuntil(b'+', drop=True)`로 `+` 직전까지 수신해 첫 번째 숫자를 추출하고,
`recvuntil(b'=?\n', drop=True)`로 두 번째 숫자를 추출한다.
합산 결과를 문자열로 인코딩해 전송하고, 50회 완료 후 `interactive()`로 플래그를 받는다.

---

## 실습 2 — 셔플된 아이템에서 flag 선택

```python
items = ["flag", "grape", "orange", "apple", "banana",
         "mango", "melon", "pineapple", "peach", "strawberry"]

for round in range(50):
    random.shuffle(items)
    for idx, item in enumerate(items):
        print(f"{idx}. {item}")
    choice = int(input("> "))
    if items[choice] != "flag":
        exit()
```

10개 아이템을 50회 셔플해서 출력하고, 10초 안에 `flag`의 인덱스를 모두 맞춰야 한다.

**풀이**

```python
import sys
from pwn import *

r = remote(sys.argv[1], int(sys.argv[2]))

for round in range(50):
    response = 0
    for i in range(10):
        if b"flag" in r.recvline():
            response = i
            break
    r.sendlineafter(b"> ", str(response).encode())

r.interactive()
```

`recvline()`으로 한 줄씩 읽으면서 `flag`가 포함된 줄의 인덱스를 `response`에 저장한다.
10개 줄을 모두 읽은 뒤 프롬프트 `> `가 나오면 해당 인덱스를 전송한다.
서버 주소와 포트를 커맨드라인 인자로 받아 유연하게 사용할 수 있다.
