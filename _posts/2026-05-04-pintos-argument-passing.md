---
title: "PintOS Argument Passing 학습 정리"
date: 2026-05-04 21:10:00 +0900
categories:
  - TIL
tags:
  - PintOS
  - OS
  - ArgumentPassing
  - Stack
---

## 오늘 다룬 내용

Week10에서는 PintOS Project 2의 `args-*` 테스트가 무엇을 검증하는지부터 다시 잡았다. 결론부터 말하면 `args-*`는 단순히 문자열을 공백 기준으로 나누는지만 보는 테스트가 아니다. 커맨드라인을 파싱한 뒤, 그 결과를 유저 프로그램이 `argc`, `argv`로 제대로 받을 수 있도록 유저 스택과 레지스터까지 올바르게 구성했는지를 확인한다.

예를 들어 입력이 다음과 같다면:

```text
args-single onearg
```

실행 파일 이름은 `args-single`이고, 유저 프로그램에 전달될 `argv`는 `["args-single", "onearg"]`가 되어야 한다. 따라서 `filesys_open()`에는 첫 번째 토큰만 들어가야 하지만, 유저 스택에는 전체 토큰 목록이 들어가야 한다.

## 실행 흐름

현재 코드 기준으로 흐름은 다음처럼 이해하면 된다.

```text
process_create_initd()
-> initd()
-> process_exec()
-> load()
-> setup_stack()
-> do_iret()
-> user program _start(argc, argv)
```

`process_exec()`는 커맨드라인 실행 요청을 받는 시작점이고, `load()`는 ELF 파일을 메모리에 올리고 유저 프로그램을 실행할 준비를 한다. `setup_stack()`은 유저 스택 페이지를 만들고, 마지막에는 `do_iret()`를 통해 유저 프로그램으로 진입한다.

중요한 점은 `load()` 안에서 실행 파일 이름과 인자 목록을 구분해야 한다는 것이다. `"args-single onearg"` 전체를 파일명으로 열면 실패한다. 파일 시스템에서 열 이름은 `args-single`이고, 나머지 인자들은 유저 스택과 레지스터로 넘겨야 한다.

## load 구현 전에 알아야 할 개념

`load()`를 이해할 때 ELF 로딩 전체를 깊게 파기보다, 지금 단계에서는 다음 정도를 우선 잡는 게 좋다.

- ELF는 실행 파일 형식이다.
- ELF header는 올바른 실행 파일인지 확인하는 정보다.
- `PT_LOAD` 타입의 segment만 실제 메모리에 올린다.
- `validate_segment()`는 유저 영역에 올려도 되는 segment인지 검사한다.
- `load_segment()`는 파일 내용을 유저 페이지에 읽고 남는 공간을 0으로 채운다.
- `install_page()`는 유저 가상 주소와 실제 물리 페이지를 페이지 테이블에 연결한다.

`args-*` 테스트에서 핵심은 ELF를 새로 구현하는 것이 아니라, ELF 로딩이 끝난 뒤 유저 프로그램이 시작할 때 필요한 스택과 레지스터를 맞추는 것이다.

## 유저 스택에 올려야 하는 것

argument passing에서 유저 스택에는 대략 다음 순서의 정보가 필요하다.

- 인자 문자열들
- 각 문자열의 주소
- `argv[argc]` 자리에 들어갈 `NULL`
- `argv[0]`, `argv[1]` 같은 문자열 주소 배열
- fake return address

x86-64에서는 함수 인자가 레지스터로 전달된다. 그래서 유저 프로그램의 `_start(int argc, char *argv[])`에 맞추려면 `argc`는 `%rdi`, `argv` 시작 주소는 `%rsi`로 들어가야 한다.

```text
argc -> rdi
argv -> rsi
```

또 하나 중요한 것은 정렬이다. `argv` 주소가 8바이트 기준으로 정렬되어 있지 않으면 `args.c` 테스트에서 실패할 수 있다. 출력이 얼추 맞아 보여도 정렬 조건에서 걸릴 수 있으니, 스택에 포인터들을 올리기 전 주소 정렬을 신경 써야 한다.

## 왜 argv 주소를 거꾸로 push하는가

오늘 가장 헷갈렸던 부분은 `argv[argc - 1]`부터 `argv[0]`까지 문자열 주소를 거꾸로 push하는 이유였다.

핵심은 스택이 낮은 주소 방향으로 자란다는 점이다. `push`는 값을 넣을 때마다 `rsp`를 감소시키고, 새 값을 더 낮은 주소에 저장한다.

최종적으로 유저 프로그램이 보는 `argv` 배열은 이렇게 보여야 한다.

```text
낮은 주소
argv[0]
argv[1]
NULL
높은 주소
```

그런데 `argv[0]`부터 순서대로 push하면 마지막에 push한 값이 가장 낮은 주소에 온다.

```text
push argv[0]
push argv[1]
push NULL
```

최종 메모리는 이렇게 되어 버린다.

```text
낮은 주소
NULL
argv[1]
argv[0]
높은 주소
```

이러면 배열의 시작 위치에 `NULL`이 와서 `argv` 순서가 망가진다. 그래서 실제로는 뒤에서부터 push해야 한다.

```text
push NULL
push argv[1]
push argv[0]
```

그러면 마지막에 push한 `argv[0]`이 가장 낮은 주소, 즉 배열의 시작 위치에 놓인다.

정리하면, 문자열의 마지막 글자부터 넣기 때문에 역순인 것이 아니다. 문자열 자체는 `"onearg\0"`처럼 원래 글자 순서 그대로 복사한다. 역순으로 push하는 대상은 문자열의 글자가 아니라 문자열 주소들이다. 마지막 인자의 주소부터 push해야 최종 `argv` 배열이 `argv[0]`, `argv[1]`, `NULL` 순서로 읽힌다.

## 오늘의 결론

PintOS argument passing의 핵심은 이 한 문장으로 정리할 수 있다.

> 파일은 첫 번째 토큰으로 열고, 전체 토큰 목록은 유저 스택과 레지스터로 넘긴다.

`args-*` 테스트를 통과하려면 파싱, 스택 성장 방향, 포인터 배열 순서, 8바이트 정렬, `%rdi`와 `%rsi` 전달 규칙을 함께 맞춰야 한다. 오늘은 정답 코드를 바로 외우기보다, 왜 이런 모양의 스택을 만들어야 하는지 이해하는 데 집중했다.
