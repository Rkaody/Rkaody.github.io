---
layout: post
title: "리눅스 사용법 한 눈에 살펴보기"
date: 2026-11-16
categories: [pwn]
tags: [Linux, Shell, Bash, 기초명령어, 권한, Dreamhack]
excerpt: "Dreamhack 리눅스 사용법 강의 정리 - 셸, 기초 명령어 17가지, 와일드카드, 리다이렉션/파이프, 권한(chmod/chown), 디렉토리 구조"
series: "Dreamhack System"
series_index: 2
---

> Dreamhack 강의 [리눅스 사용법 한 눈에 살펴보기](https://learn.dreamhack.io/441) 정리 (Ubuntu 22.04 기준)

## 셸(Shell)

셸(Shell)은 사용자가 운영체제와 상호작용하는 인터페이스다. 리눅스에서는 주로 **bash(Bourne Again Shell)** 를 사용한다.

셸 프롬프트는 다음과 같은 형식으로 표시된다:

```
user@user-VirtualBox:~$
```

- `user` : 현재 로그인한 사용자 이름
- `user-VirtualBox` : 호스트(컴퓨터) 이름
- `~` : 현재 디렉토리 (`~`는 홈 디렉토리를 의미)
- `$` : 일반 사용자 프롬프트 (`#`이면 root)

---

## 기초적인 명령어

### 1. sudo apt update

설치 가능한 소프트웨어 패키지 목록을 업데이트한다.

```bash
user@user-VirtualBox:~$ sudo apt update
```

### 2. sudo apt upgrade

설치된 소프트웨어 패키지를 최신 버전으로 업그레이드한다. 오래된 패키지에는 보안 취약점이 존재할 수 있으므로 주기적으로 실행하는 것이 권장된다.

```bash
user@user-VirtualBox:~$ sudo apt upgrade
```

### 3. id

현재 유저의 UID(User ID)와 소속 GID(Group ID)를 출력한다. 리눅스는 권한 기반으로 파일에 접근하므로, 자신의 권한을 확인할 때 사용한다.

```bash
user@user-VirtualBox:~$ id
uid=1000(user) gid=1000(user) groups=1000(user),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),122(lpadmin),134(lxd),135(sambashare)
```

### 4. pwd

**P**rint **W**orking **D**irectory. 현재 작업 중인 디렉토리 경로를 출력한다.

```bash
user@user-VirtualBox:~$ pwd
/home/user
```

### 5. ls

**L**i**s**t. 디렉토리 내용을 출력한다. `-l` 플래그를 추가하면 권한, 소유자, 크기 등 상세 정보를 함께 출력한다.

```bash
user@user-VirtualBox:~$ ls
Desktop  Documents  Downloads  Music  Pictures  Public  snap  Templates  Videos

user@user-VirtualBox:~$ ls -l
total 36
drwxr-xr-x 2 user user 4096 11월 30 18:46 Desktop
drwxr-xr-x 2 user user 4096 11월 30 18:46 Documents
...
```

다른 디렉토리 내용도 볼 수 있다: `ls -l /`

### 6. cd

**C**hange **D**irectory. 작업 디렉토리를 변경한다.

**절대 경로(Absolute Path)**: 루트 `/`부터 시작하는 전체 경로 (예: `/home/user`)

**상대 경로(Relative Path)**: 현재 위치 기준의 경로 (예: `..`은 부모 디렉토리)

특수 경로:
- `~` : 현재 유저의 홈 디렉토리
- `-` : 이전에 위치했던 디렉토리
- `..` : 부모 디렉토리

```bash
user@user-VirtualBox:~$ cd /         # 루트로 이동
user@user-VirtualBox:/$ cd /home/user # 절대경로로 이동
user@user-VirtualBox:~$ cd ..         # 부모 디렉토리로 이동
user@user-VirtualBox:/home$ cd ~      # 홈 디렉토리로 이동
```

### 7. mkdir

**M**a**k**e **Dir**ectory. 디렉토리를 생성한다.

```bash
user@user-VirtualBox:~$ mkdir new_dir
```

### 8. touch

비어 있는 새 파일을 생성한다.

```bash
user@user-VirtualBox:~/new_dir$ touch new_file
user@user-VirtualBox:~/new_dir$ ls -l
total 0
-rw-rw-r-- 1 user user 0 12월  1 20:59 new_file
```

### 9. mv

**M**o**v**e. 파일이나 디렉토리를 이동하거나 이름을 변경한다.

```bash
# 이름 변경
user@user-VirtualBox:~/new_dir$ mv new_file old_file

# 파일 이동 (상위 디렉토리로)
user@user-VirtualBox:~/new_dir$ mv old_file ..
```

### 10. rm

**R**e**m**ove. 파일을 삭제한다. 디렉토리는 `-r` 플래그로 재귀적으로 삭제한다.

```bash
user@user-VirtualBox:~$ rm file.txt
user@user-VirtualBox:~$ rm -r dir/
```

### 11. cat

파일 내용을 출력한다. 주로 텍스트 파일을 빠르게 확인할 때 사용한다.

```bash
user@user-VirtualBox:~$ cat /etc/passwd
root:x:0:0:root:/root:/bin/bash
...
user:x:1000:1000:user,,,:/home/user:/bin/bash
```

### 12. file

파일의 타입(형식)을 출력한다. 확장자가 없는 바이너리의 종류를 파악할 때 유용하다.

```bash
user@user-VirtualBox:~$ file /bin/ls
/bin/ls: ELF 64-bit LSB pie executable, x86-64, ...
```

### 13. echo

문자열을 출력한다. 리다이렉션(`>`)과 함께 파일에 내용을 쓸 수 있다.

```bash
user@user-VirtualBox:~$ echo "Hello, World!"
Hello, World!

user@user-VirtualBox:~$ echo "Hello" > hello.txt
```

### 14. cp

**C**o**p**y. 파일이나 디렉토리를 복사한다. 디렉토리는 `-r` 플래그가 필요하다.

```bash
user@user-VirtualBox:~$ cp file.txt file_copy.txt
user@user-VirtualBox:~$ cp -r dir/ dir_copy/
```

### 15. grep

파일 내에서 특정 문자열을 검색한다.

```bash
user@user-VirtualBox:~$ grep "root" /etc/passwd
root:x:0:0:root:/root:/bin/bash
```

### 16. man

**Man**ual. 명령어의 사용법(매뉴얼)을 출력한다.

```bash
user@user-VirtualBox:~$ man ls
```

### 17. curl

URL로부터 데이터를 전송하거나 받아오는 명령어.

| 옵션 | 설명 |
|------|------|
| `-o` | 응답을 파일로 저장 |
| `-i` | 응답 헤더 포함 출력 |
| `-X` | HTTP 메서드 지정 (GET, POST 등) |
| `-d` | POST 데이터 전송 |

```bash
# GET 요청
user@user-VirtualBox:~$ curl https://example.com

# POST 요청
user@user-VirtualBox:~$ curl -X POST https://example.com -d "data=value"

# CTF에서 자주 쓰는 패턴 (파일 내용을 서버로 전송)
user@user-VirtualBox:~$ curl URL -d "$(cat /etc/passwd)"
```

---

## 와일드카드

셸에서 파일 이름 패턴 매칭에 사용하는 특수 문자들이다.

| 와일드카드 | 의미 | 예시 |
|-----------|------|------|
| `?` | 임의의 문자 1개 | `file?.txt` → `file1.txt`, `fileA.txt` |
| `*` | 임의의 문자 0개 이상 | `*.txt` → 모든 .txt 파일 |
| `[범위]` | 범위 내 문자 1개 | `test[0-9]` → `test0`~`test9` |

```bash
user@user-VirtualBox:~/new_dir$ ls test[0-9]
test1  test2  test3
```

---

## 리다이렉션과 파이프

### 리다이렉션(Redirection)

표준 입출력의 방향을 변경하는 작업이다.

**`명령어 > 파일`** : 표준 출력을 파일로 저장 (덮어쓰기)

```bash
user@user-VirtualBox:~/new_dir$ ls test[0-9] > world
user@user-VirtualBox:~/new_dir$ cat world
test1
test2
test3
```

**`명령어 >> 파일`** : 표준 출력을 파일에 추가 (이어쓰기)

```bash
user@user-VirtualBox:~/new_dir$ cat hello >> world
user@user-VirtualBox:~/new_dir$ cat world
test1
test2
test3
hello
```

**`명령어 < 파일`** : 파일 내용을 표준 입력으로 사용

```bash
user@user-VirtualBox:~/new_dir$ grep test < world
test1
test2
test3
```

### 파이프(Pipe)

`|` 문자를 사용해 앞 명령어의 표준 출력을 뒷 명령어의 표준 입력으로 전달한다.

```bash
user@user-VirtualBox:~/new_dir$ ls -l | grep hello
-rw-rw-r-- 1 user user 13 12월  2 13:05 hello
```

---

## 권한(Permission)

### 유저(User)와 그룹(Group)

리눅스의 모든 유저는 고유한 **UID(User ID)** 를 가지며, 그룹은 고유한 **GID(Group ID)** 를 가진다. 파일 접근 시 UID와 GID로 권한을 검사한다.

**`/etc/passwd`** : 유저 정보 파일

```
root:x:0:0:root:/root:/bin/bash
user:x:1000:1000:user,,,:/home/user:/bin/bash
```

형식: `사용자명:암호:UID:GID:설명:홈디렉토리:로그인셸`

**`/etc/group`** : 그룹 정보 파일

```
root:x:0:
user:x:1000:
sudo:x:27:user
```

### 파일 및 디렉토리 권한

리눅스의 모든 파일/디렉토리는 **소유자(owner)** 와 **소유 그룹(group)** 을 가진다. 세 가지 접근 권한이 있다:

- **읽기(Read, r)** : 파일/디렉토리 내용을 볼 수 있음
- **쓰기(Write, w)** : 파일/디렉토리 내용을 수정하거나 삭제할 수 있음
- **실행(Execute, x)** : 파일을 실행하거나, 디렉토리 내용에 접근할 수 있음

`ls -l` 출력 형식:

```bash
user@user-VirtualBox:~/new_dir$ ls -l
total 12
drwxrwxr-x 2 user user 4096 12월  2 13:38 dir
---------- 1 user user   13 12월  2 13:05 hello
-rwxrw-r-- 1 user user   13 12월  2 13:08 world
```

### 권한 플래그 해석

`drwxrwxr-x` 를 분해하면:

```
d  rwx  rwx  r-x
│   │    │    └─ 기타(others) 권한
│   │    └────── 소유 그룹(group) 권한
│   └─────────── 소유자(owner) 권한
└─────────────── 파일 타입 (d=디렉토리, -=일반파일, l=링크)
```

| 플래그 | 2진수 | 8진수(10진수) |
|--------|-------|--------------|
| `rwx`  | `111` | `7` |
| `rw-`  | `110` | `6` |
| `r-x`  | `101` | `5` |
| `r--`  | `100` | `4` |
| `---`  | `000` | `0` |

**예시 해석**: `-rwxrw-r-- 1 user user 13 world`
- 일반 파일 (`-`)
- 소유자(`user`): 읽기/쓰기/실행 가능 (`rwx`)
- 소유 그룹(`user`): 읽기/쓰기 가능, 실행 불가 (`rw-`)
- 기타 유저: 읽기만 가능 (`r--`)

### chmod

파일 권한을 변경하는 명령어. root 또는 파일 소유자만 실행 가능하다.

**숫자(8진수) 방식:**

```bash
# rwx=7, rw-=6, r--=4 → chmod 764
user@user-VirtualBox:~/new_dir$ chmod 764 hello
user@user-VirtualBox:~/new_dir$ ls -l
-rwxrw-r-- 1 user user   13 12월  2 13:05 hello
```

**기호(symbolic) 방식:**

```bash
# 소유 그룹에 실행 권한 추가
user@user-VirtualBox:~/new_dir$ chmod g+x hello

# 소유 그룹에서 쓰기/실행 권한 제거
user@user-VirtualBox:~/new_dir$ chmod g-wx hello
```

기호 대상: `u`(소유자), `g`(소유 그룹), `o`(기타), `a`(전체)
연산자: `+`(추가), `-`(제거), `=`(지정)

### chown

파일 소유자 또는 소유 그룹을 변경하는 명령어. **root만** 실행 가능하다.

```bash
# 소유자를 root로 변경
user@user-VirtualBox:~/new_dir$ sudo chown root hello
user@user-VirtualBox:~/new_dir$ ls -l
-rwxr--r-- 1 root user   13 12월  2 13:05 hello

# 이제 user는 hello 파일을 수정할 수 없음
user@user-VirtualBox:~/new_dir$ echo "hello" > hello
bash: hello: Permission denied
```

소유 그룹만 변경하려면 `chgrp` 명령어를 사용한다.

---

## 디렉토리 구조

리눅스의 루트 디렉토리 `/` 아래에는 목적별로 구분된 표준 디렉토리들이 있다.

| 디렉토리 | 설명 |
|----------|------|
| `/bin` | 기본 명령어 바이너리 (ls, cd, cat 등) |
| `/boot` | 부팅에 필요한 파일 (커널 이미지 등) |
| `/dev` | 장치(Device) 파일 |
| `/etc` | 시스템 설정 파일 (passwd, group 등) |
| `/home` | 일반 유저의 홈 디렉토리 |
| `/lib` | 공유 라이브러리 |
| `/opt` | 추가 소프트웨어 패키지 |
| `/proc` | 프로세스 정보 (가상 파일시스템) |
| `/root` | root 유저의 홈 디렉토리 |
| `/sbin` | 시스템 관리용 바이너리 (root 전용) |
| `/tmp` | 임시 파일 저장소 (재부팅 시 삭제) |
| `/usr` | 사용자 프로그램 및 데이터 |
| `/var` | 가변 데이터 (로그, 캐시 등) |

```bash
user@user-VirtualBox:~$ ls -l /
lrwxrwxrwx   1 root root    7 bin -> usr/bin
drwxr-xr-x   4 root root 4096 boot
drwxr-xr-x  19 root root 4180 dev
drwxr-xr-x 130 root root 12288 etc
drwxr-xr-x   3 root root 4096 home
...
```

---

## 정리

이 강의에서 다룬 핵심 내용:

- **셸**: 리눅스와 사용자 사이의 인터페이스, bash 사용
- **기초 명령어 17가지**: 패키지 관리(`apt`), 파일/디렉토리 조작(`ls`, `cd`, `mkdir`, `touch`, `mv`, `rm`, `cp`), 내용 확인(`cat`, `file`, `grep`), 도움말(`man`), 네트워크(`curl`)
- **와일드카드**: `?`, `*`, `[범위]`로 파일 패턴 매칭
- **리다이렉션/파이프**: `>`, `>>`, `<`, `|`로 입출력 방향 제어
- **권한**: UID/GID 기반 접근 제어, `chmod`로 권한 변경, `chown`으로 소유자 변경
- **디렉토리 구조**: 목적별로 구분된 표준 디렉토리 구조 이해
