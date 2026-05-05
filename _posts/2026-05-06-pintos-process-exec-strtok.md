---
title: "PintOS process_exec와 strtok_r 흐름 정리"
date: 2026-05-06 01:30:00 +0900
categories:
  - TIL
tags:
  - PintOS
  - OS
  - C
  - Syscall
  - ArgumentPassing
---

## 오늘 다룬 내용

오늘은 PintOS Project 2에서 커맨드라인 인자를 파싱하는 과정과 `process_exec()`, `load()` 사이에서 값이 어떻게 넘어가는지를 정리했다. 이전에는 argument passing의 최종 스택 모양을 중심으로 봤다면, 오늘은 그 앞 단계인 문자열 파싱, `argv` 배열 구성, 디버그 출력, 함수 반환 흐름을 더 자세히 봤다.

핵심 질문은 다음과 같았다.

- `strtok_r`의 `r`은 무엇인가?
- `saveptr`는 어디서 선언하고 어떤 역할을 하는가?
- `strtok_r`는 토큰을 배열로 저장해주는가?
- `argv`는 왜 `char *argv[]` 또는 `char **argv`로 표현되는가?
- `process_exec()`의 `f_name`은 어디서 오고, `load()`의 `file_name`은 무엇을 받는가?
- `load()`가 `success` 값을 어떻게 `process_exec()`에 돌려주는가?

## strtok_r 이해하기

`strtok_r`의 `r`은 `reentrant`를 뜻한다. 일반 `strtok`은 내부 static 상태를 사용하기 때문에 중첩 호출이나 멀티스레드 상황에서 꼬일 수 있다. 반면 `strtok_r`는 현재 어디까지 파싱했는지를 호출자가 넘긴 `saveptr`에 저장한다.

```c
char *saveptr;
char *token = strtok_r(input, " ", &saveptr);
```

여기서 각 인자의 역할은 다음과 같다.

- 첫 번째 인자 `input`: 처음 파싱할 원본 문자열
- 두 번째 인자 `" "`: 공백을 구분자로 사용한다는 뜻
- 세 번째 인자 `&saveptr`: 다음 파싱 위치를 저장할 포인터의 주소
- 반환값 `token`: 이번에 잘라낸 토큰의 시작 주소

두 번째 토큰부터는 첫 번째 인자에 `NULL`을 넣는다.

```c
token = strtok_r(NULL, " ", &saveptr);
```

이때 `strtok_r`가 이어서 어디부터 잘라야 하는지는 `saveptr`가 기억하고 있다.

중요한 점은 `strtok_r`가 원본 문자열을 직접 수정한다는 것이다. 예를 들어 `"echo hello world"`를 공백 기준으로 자르면 내부적으로는 다음처럼 바뀐다.

```text
echo\0hello\0world\0
```

따라서 원본 커맨드라인을 두 번 파싱해야 한다면 같은 버퍼 하나를 재사용하면 안 된다. 한 번 토큰화된 문자열은 첫 번째 `\0` 이후가 일반 문자열 출력에서 보이지 않기 때문이다.

## strtok_r는 배열을 만들지 않는다

처음에는 `strtok_r[0] = echo`, `strtok_r[1] = hello`처럼 저장되는지 헷갈렸다. 하지만 `strtok_r`는 토큰 배열을 만들어주지 않는다. 원본 문자열 안의 구분자를 `\0`로 바꾸고, 매번 이번 토큰의 시작 주소만 반환한다.

즉 다음과 같은 배열은 직접 만들어야 한다.

```text
argv[0] -> "echo"
argv[1] -> "hello"
argv[2] -> "world"
argv[3] -> NULL
```

`argv`는 문자열 자체를 복사해서 담는 배열이 아니라, 문자열 시작 주소들을 담는 포인터 배열이다. 그래서 `argv[0]`, `argv[1]`의 타입은 각각 `char *`가 된다.

```c
char *argv[4];
```

이 선언은 "`argv`는 배열이고, 각 원소는 `char *`이다"라는 뜻이다.

함수 인자로 넘길 때 `char **argv`가 나오는 이유도 여기에 있다. 배열 이름은 식에서 첫 번째 원소의 주소처럼 동작하고, 첫 번째 원소의 타입이 `char *`이므로 그 주소의 타입은 `char **`가 된다.

```text
argv
 |
 v
+---------+       +-------------+
| argv[0] | ----> | echo\0      |
+---------+       +-------------+
| argv[1] | ----> | hello\0     |
+---------+       +-------------+
| argv[2] | ----> | world\0     |
+---------+       +-------------+
| argv[3] | ----> | NULL        |
+---------+
```

## argv 크기를 정하는 방법

`argv`를 `argc`에 맞춰 딱 만들고 싶다면 먼저 `argc`를 알아야 한다. 그래서 보통은 토큰 개수를 먼저 세고, 그 다음 배열을 만든다.

방법은 크게 두 가지로 나눌 수 있다.

- `char *argv[argc + 1]`: 함수 안에서 잠깐 쓰는 VLA 방식
- `malloc(sizeof(char *) * (argc + 1))`: 힙에 잡고 직접 `free`하는 방식

`argv[argc + 1]`만 해도 크기는 확보된다. 다만 이 배열은 현재 함수의 스택에 잡히기 때문에 함수가 끝나면 사라진다. PintOS 커널 스택은 크기가 작으므로 큰 배열을 스택에 잡는 것도 조심해야 한다.

마지막에 `+1`을 하는 이유는 `argv[argc] = NULL` 자리가 필요하기 때문이다. 이 `NULL`은 인자 배열의 끝을 표시한다.

## process_exec의 f_name은 어디서 오는가

`process_exec(void *f_name)`의 `f_name`은 테스트 C 코드가 직접 넘기는 값이라기보다는, PintOS 실행 명령에서 들어온 커맨드라인 문자열이 커널 내부 경로를 거쳐 전달된 값이다.

초기 실행 흐름은 대략 다음과 같다.

```text
threads/init.c의 run_task()
-> process_create_initd(task)
-> thread_create(..., initd, fn_copy)
-> initd(void *f_name)
-> process_exec(f_name)
```

예를 들어 테스트 실행 명령이 `run 'args-single onearg'`라면, 이 문자열이 복사되어 새 스레드의 인자로 넘어가고 결국 `process_exec(f_name)`으로 들어온다.

`process_exec()` 안에서는 보통 다음처럼 받는다.

```c
char *file_name = f_name;
```

이후 `load(file_name, &_if)`를 호출하면, `load()`의 `file_name` 매개변수는 `process_exec()`에서 넘긴 값을 받는다.

```text
process_exec의 file_name
-> load의 file_name
```

만약 `process_exec()`에서 먼저 `strtok_r(file_name, " ", &saveptr)`를 호출했다면 원본 문자열은 `"args-single\0onearg"`처럼 바뀐다. 그러면 `load(file_name, &_if)`에 넘겼을 때 `load()` 입장에서는 `file_name`이 `"args-single"`처럼 보인다.

이 방식은 실행 파일 이름만 열어야 하는 상황에서는 도움이 되지만, 전체 인자 목록을 나중에 다시 써야 한다면 별도 복사본이나 이미 만든 `argv`, `argc`를 보존해야 한다.

## 디버그 출력에서 배운 점

파싱 결과를 확인하려고 `printf`를 넣을 때도 타입을 맞춰야 한다.

```c
printf("argc = %d\n", argc);
printf("argv[%d] = %s\n", index, argv[index]);
```

- `%d`: 정수 출력
- `%s`: `char *`가 가리키는 문자열 출력

`printf("argv[%d] = %s\n");`처럼 포맷만 쓰고 실제 값을 넘기지 않으면 위험하다. `printf`는 포맷 문자열을 믿고 인자를 해석하기 때문에, 인자가 없거나 타입이 맞지 않으면 이상한 값이 나오거나 커널이 터질 수 있다.

또 `NULL` 확인을 `%s`로 할 수도 있지만, 더 명확하게 보려면 비교 결과를 출력하는 편이 낫다.

```text
argv[argc] == NULL 이면 1
아니면 0
```

그리고 줄바꿈이 없으면 다음 출력이 바로 붙어서 헷갈릴 수 있다. 예를 들어 `null_check : (null)(args) begin`처럼 보인다면 `NULL` 뒤에 이상한 값이 들어간 것이 아니라, `\n` 없이 다음 출력이 붙은 것일 수 있다.

## 테스트와 process_wait

`args-*` 테스트를 끝까지 PASS/FAIL로 확인하려면 `process_wait()`도 정상적으로 작동해야 한다. 만약 `process_wait()`가 아직 무한루프라면 테스트가 timeout으로 끝날 수 있다.

그래서 상태를 나눠서 봐야 한다.

- 컴파일 확인: argument passing 완성 전에도 가능
- 파싱 중간 확인: 디버그 출력으로 `argv[0]`, `argv[1]` 확인 가능
- `args-*` 테스트 통과 확인: 유저 스택 구성과 `process_wait()`까지 필요

디버그 출력은 중간 확인에는 유용하지만, 정식 테스트 전에는 지워야 한다. PintOS 테스트는 출력 비교가 엄격해서 불필요한 출력이 남아 있으면 실패할 수 있다.

## fake return address

argument passing에서 fake return address는 실제로 돌아갈 주소라기보다, 유저 프로그램이 보통 함수 호출로 시작한 것처럼 스택 모양을 맞추기 위한 자리 채움 값이다.

PintOS의 유저 프로그램 시작점은 대략 다음과 같다.

```c
_start(argc, argv)
```

그리고 `_start()`는 내부에서 `main(argc, argv)`를 호출한 뒤 `exit()`로 종료한다. 정상 흐름에서는 fake return address를 실제로 사용하지 않는다.

그럼에도 fake return address를 넣는 이유는 함수 호출 규약에 맞는 스택 모양을 만들어주기 위해서다. 64비트 환경에서는 주소 크기에 맞춰 보통 8바이트 값을 넣고, 실제 돌아갈 곳이 없다는 의미로 `0`을 둔다.

## load의 success 반환 흐름

마지막으로 헷갈렸던 부분은 `success = load(file_name, &_if);`가 어떻게 값을 받는지였다.

`process_exec()` 안의 `success`와 `load()` 안의 `success`는 이름이 같아도 서로 다른 지역 변수다. `load()`는 내부에서 성공 여부를 계산하고, 마지막에 그 값을 반환한다.

```text
load 내부 success
-> return success
-> process_exec의 success에 저장
```

즉 `success = true;`만으로 값이 밖으로 나가는 것이 아니라, `load()` 마지막의 `return success;`가 있어야 호출자에게 전달된다.

반면 `process_exec()`의 성공 경로는 일반적인 `return`으로 끝나지 않는다. 성공하면 `do_iret(&_if)`로 유저 모드 실행 상태로 전환하고, 실패할 때만 `return -1`로 호출자에게 실패를 알린다.

## 오늘의 결론

오늘 배운 핵심은 `strtok_r`로 문자열을 나누는 일과 PintOS의 프로세스 실행 흐름이 서로 맞물려 있다는 점이다.

`strtok_r`는 원본 문자열을 직접 바꾸고, 토큰 배열을 자동으로 만들어주지 않는다. `argv`는 문자열 포인터 배열이고, `process_exec()`에서 받은 커맨드라인은 `load()`로 실행 파일 이름을 넘기는 동시에 유저 스택에 올릴 인자 정보로도 이어져야 한다.

정리하면, argument passing 구현은 단순 파싱 문제가 아니라 다음 흐름을 끝까지 연결하는 작업이다.

```text
커맨드라인 문자열
-> strtok_r로 토큰 분리
-> argc, argv 구성
-> 실행 파일 이름으로 load
-> 유저 스택에 인자 배치
-> rdi, rsi 설정
-> do_iret로 유저 프로그램 시작
```

오늘은 이 흐름 중에서 특히 `strtok_r`, `argv`, `process_exec()`, `load()`의 값 전달 관계를 정리했다.
