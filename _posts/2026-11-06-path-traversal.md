---
layout: post
title: "Path Traversal — 경로 순회 취약점"
date: 2026-11-06
categories: [pwn]
tags: [linux, c, dreamhack, path-traversal]
excerpt: "절대/상대 경로 개념과 Path Traversal 취약점 원리, 취약한 C 코드 분석 및 방어 기법 정리."
series: "취약점 분석"
series_index: 1
---

로컬 파일 시스템에 접근하는 서비스를 외부에 공개할 때는 접근 가능한 경로에 반드시 제한을 두어야 한다. 사용자 입력을 검증 없이 파일 경로에 사용하면, 허용되지 않은 경로까지 접근할 수 있는 **Path Traversal** 취약점이 발생한다. 임의 파일 읽기/쓰기의 수단으로 활용되며, 계정 정보 탈취나 서버 장악으로 이어질 수 있다.

## 경로 개념

### 절대 경로

루트 디렉토리(`/`)부터 대상 파일까지 거치는 모든 디렉토리를 `/`로 연결한 경로다. 한 파일에 대응되는 절대 경로는 유일하며, 현재 위치와 무관하게 동일한 파일을 가리킨다.

```
/
└── a
    └── b
        ├── c
        │   └── target
        └── d  ← 현재 경로
```

위 구조에서 `target`의 절대 경로: `/a/b/c/target`

### 상대 경로

현재 디렉토리를 기준으로 대상 파일에 이르는 경로다. `..`는 상위 디렉토리, `.`는 현재 디렉토리를 의미한다. 한 파일을 가리키는 상대 경로는 무수히 많다.

현재 위치가 `d`(`/a/b/d`)일 때 `target`을 가리키는 상대 경로:

- `../c/target`
- `../../b/c/target`
- `./../../b/c/target`

Path Traversal은 이 `..`를 이용해 허용된 디렉토리 밖으로 탈출하는 공격이다.

---

## 취약한 코드 분석

```c
// Name: path_traversal.c
// Compile: gcc -o path_traversal path_traversal.c

#include <stdio.h>
#include <stdlib.h>
#include <string.h>

const int kMaxNameLen = 0x100;
const int kMaxPathLen = 0x200;
const int kMaxDataLen = 0x1000;
const char *kBasepath = "/tmp";

int main() {
    char file_name[kMaxNameLen];
    char file_path[kMaxPathLen];
    char data[kMaxDataLen];
    FILE *fp = NULL;  // FILE: 파일 스트림 정보를 담는 구조체

    memset(file_name, '\0', kMaxNameLen);
    memset(file_path, '\0', kMaxPathLen);
    memset(data, '\0', kMaxDataLen);

    printf("File name: ");
    fgets(file_name, kMaxNameLen, stdin);

    // fgets는 개행 문자까지 저장하므로 제거
    file_name[strcspn(file_name, "\n")] = '\0';

    // 취약점 발생 지점: 입력값을 검증 없이 경로에 직접 삽입
    snprintf(file_path, kMaxPathLen, "%s/%s", kBasepath, file_name);

    if ((fp = fopen(file_path, "r")) == NULL) {
        fprintf(stderr, "No file named %s", file_name);
        return -1;
    }

    fgets(data, kMaxDataLen, fp);
    printf("%s", data);

    fclose(fp);
    return 0;
}
```

### 취약점 발생 지점

```c
snprintf(file_path, kMaxPathLen, "%s/%s", kBasepath, file_name);
```

개발자의 의도는 `/tmp` 하위 파일만 읽는 것이지만, 입력값에 `..`가 포함되면 상위 디렉토리로 탈출할 수 있다.

```
입력: ../etc/passwd

/tmp  +  /  +  ../etc/passwd
= /tmp/../etc/passwd
= /etc/passwd
```

{% include callout.html type="danger" content="`snprintf`는 경로를 그대로 이어 붙일 뿐 `..`를 해석하지 않는다. 사용자 입력이 검증 없이 경로에 삽입되면 기준 디렉토리 밖의 임의 파일에 접근할 수 있다." %}

### 실행 흐름

```
사용자 입력 (fgets)
      ↓
\n 제거 (strcspn)
      ↓
경로 생성 (snprintf): /tmp/ + 입력값
      ↓
fopen 시도
      ├─ 정상 입력 (hello.txt)
      │   → /tmp/hello.txt 열기 → 내용 출력 ✅
      │
      ├─ 존재하지 않는 파일
      │   → fp = NULL → 에러 출력 후 종료 ❌
      │
      ├─ Path Traversal 성공 (../etc/passwd)
      │   → /etc/passwd 접근 → 계정 목록 출력 🚨
      │
      └─ Path Traversal 실패 (../etc/shadow)
          → /etc/shadow 접근 시도
          → root 전용 파일 (권한: rw-r-----) → fp = NULL ❌
```

`/etc/passwd`는 모든 사용자가 읽을 수 있는 반면, `/etc/shadow`는 root만 읽을 수 있어 일반 프로세스로는 접근이 차단된다.

{% include callout.html type="warning" content="root 권한으로 실행되는 서비스(예: setuid 바이너리, 특정 데몬)에서 같은 취약점이 발생하면 `/etc/shadow`를 포함한 모든 파일에 접근 가능하다. 취약점의 영향 범위는 프로세스 권한에 비례한다." %}

---

## 사용 함수 정리

### string.h

| 함수 | 설명 |
|------|------|
| `memset(ptr, val, size)` | `ptr`부터 `size` 바이트를 `val`로 초기화. 선언 직후 쓰레기값 제거에 사용 |
| `strcspn(str, reject)` | `str`에서 `reject`에 포함된 문자가 처음 나타나는 위치 반환 |
| `strstr(haystack, needle)` | `haystack`에서 `needle` 문자열을 찾아 포인터 반환. 없으면 `NULL` |

### stdio.h

| 함수 | 설명 |
|------|------|
| `printf(fmt, ...)` | 표준 출력(stdout)에 출력 |
| `fprintf(stream, fmt, ...)` | 지정한 스트림에 출력. `stderr`에 쓰면 에러 메시지 출력 |
| `fgets(buf, size, stream)` | 스트림에서 `size-1`바이트 또는 `\n`까지 읽어 `buf`에 저장 |
| `snprintf(buf, size, fmt, ...)` | 형식에 맞는 문자열을 최대 `size-1`바이트만큼 `buf`에 저장 |
| `fopen(path, mode)` | 파일 열기. `"r"` 읽기, `"w"` 쓰기, `"a"` 추가. 실패 시 `NULL` 반환 |
| `fclose(fp)` | 열린 파일 스트림 닫기. 미호출 시 메모리 누수 발생 |

---

## 방어 기법

### 1. `realpath()`로 정규 경로 검증

```c
#include <stdlib.h>

char resolved[kMaxPathLen];

// 심볼릭 링크와 ..를 모두 해석한 절대 경로를 반환
if (realpath(file_path, resolved) == NULL) {
    // 경로 해석 실패
    return -1;
}

// 허용된 기본 경로로 시작하는지 확인
if (strncmp(resolved, kBasepath, strlen(kBasepath)) != 0) {
    fprintf(stderr, "Access denied\n");
    return -1;
}
```

가장 권장되는 방어 방법이다. `..`, 심볼릭 링크, 중복 슬래시 등을 모두 해석한 뒤 기준 경로로 시작하는지 확인한다.

{% include callout.html type="tip" content="`realpath()`는 경로의 대상 파일이 **실제로 존재하지 않으면 `NULL`을 반환**한다. 아직 생성되지 않은 파일 경로를 검증할 때는 상위 디렉토리까지만 `realpath()`로 확인하고 파일명은 별도로 화이트리스트 검증한다." %}

### 2. `strstr()`로 `..` 탐지

```c
if (strstr(file_name, "..") != NULL) {
    fprintf(stderr, "Invalid file name\n");
    return -1;
}
```

단순하지만 우회 가능성이 있다. `..`를 URL 인코딩(`%2e%2e`)하거나 null 바이트를 삽입하는 등의 방법으로 우회될 수 있어 단독으로 사용하기엔 부족하다.

{% include callout.html type="warning" content="URL 디코딩이 경로 처리 전에 이루어지는 환경(웹 프레임워크, Nginx, Apache 등)에서는 `%2e%2e%2f`→`../`와 같이 인코딩된 시퀀스가 `strstr` 검사를 우회할 수 있다. `strstr` 단독 방어는 신뢰하지 않는다." %}

### 3. 화이트리스트 검증

허용할 파일명 패턴을 명시적으로 정의하고, 그 외 입력은 모두 거부하는 방식이다. 알파벳, 숫자, `.`, `-`만 허용하는 식으로 구현할 수 있다.

---

Path Traversal은 입력값을 파일 경로에 사용할 때 반드시 고려해야 하는 취약점이다. 방어의 핵심은 `realpath()`로 경로를 정규화한 뒤 기준 디렉토리 내에 있는지 확인하는 것이다.
