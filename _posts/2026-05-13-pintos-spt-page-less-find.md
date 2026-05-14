---
title: "PintOS VM: page_less와 spt_find_page 흐름 이해"
date: 2026-05-13 23:50:00 +0900
categories:
  - TIL
tags:
  - PintOS
  - OS
  - VirtualMemory
  - SPT
  - HashTable
  - C
---

## 오늘 다룬 내용

오늘은 PintOS Project 3 VM에서 supplemental page table(SPT)을 hash table로 구현할 때, `page_less()`가 왜 비교를 하는지와 `spt_find_page()`가 어떤 흐름으로 동작해야 하는지를 정리했다.

이전에는 SPT가 "가상주소 `va`를 key로 해서 `struct page` metadata를 찾는 per-process table"이라는 큰 구조를 잡았다. 오늘은 그 구조가 실제 PintOS hash API 안에서 어떻게 동작하는지 더 좁혀서 봤다.

핵심 질문은 다음과 같았다.

- `void *` 비교와 `unsigned *` 비교는 무엇이 다른가?
- `page_less()`는 왜 `==`가 아니라 `<`를 쓰는가?
- PintOS hash table은 같은 key인지 어떻게 판단하는가?
- `hash_entry()`는 `hash_elem`에서 어떻게 `struct page *`를 꺼내는가?
- `spt_find_page()`에서 왜 `pg_round_down()`을 써야 하는가?
- `pintos2` 기준으로 SPT 구현 1, 2번은 무엇을 해야 하는가?

## 포인터 비교 정리

`void *`는 타입을 모르는 주소다. 주소가 같은지 비교하는 것은 가능하지만, 그 주소가 가리키는 값을 직접 역참조할 수는 없다.

```c
void *a;
void *b;

if (a == b) {
    /* 같은 주소인지 비교 */
}
```

반면 `unsigned *`는 "unsigned int를 가리키는 포인터"다. 여기서 `unsigned`는 주소 자체가 아니라, 포인터가 가리키는 값의 타입에 붙는다.

```c
unsigned *a;
unsigned *b;

if (*a < *b) {
    /* a와 b가 가리키는 unsigned 값을 비교 */
}
```

포인터 자체를 숫자처럼 비교해야 한다면 `unsigned`로 캐스팅하는 것은 위험하다. 64비트 주소를 32비트 `unsigned`에 담으면 주소가 잘릴 수 있기 때문이다. 주소를 정수로 다루는 의도를 명확히 하려면 `uintptr_t`가 맞다.

```c
#include <stdint.h>

if ((uintptr_t) a < (uintptr_t) b) {
    /* 주소값 비교 */
}
```

PintOS 코드에서 `va`를 비교할 때 `uint64_t`로 캐스팅하는 것은 "가상주소 값을 기준으로 순서를 비교한다"는 의미로 볼 수 있다.

## page_less가 비교하는 이유

`page_less()`는 페이지의 우선순위를 정하려는 함수가 아니다. SPT hash table에서 page를 `va` 기준으로 구분하기 위한 비교 함수다.

```c
static bool
page_less (const struct hash_elem *a,
           const struct hash_elem *b,
           void *aux UNUSED) {
    const struct page *page_a = hash_entry (a, struct page, hash_elem);
    const struct page *page_b = hash_entry (b, struct page, hash_elem);

    return (uint64_t) page_a->va < (uint64_t) page_b->va;
}
```

여기서 `a`와 `b`는 같은 데이터를 뜻하지 않는다. 둘 다 `struct hash_elem *` 타입일 뿐이고, 실제로는 서로 다른 `struct page` 안에 들어 있는 `hash_elem`일 수 있다.

```text
a = &page1.hash_elem
b = &page2.hash_elem
```

`hash_entry()`는 이 `hash_elem` 주소를 이용해 원래의 `struct page *`를 복구한다. 따라서 최종 비교는 다음과 같다.

```c
page1.va < page2.va
```

반환값은 `bool`이다.

```text
true  = page_a->va가 page_b->va보다 작다
false = page_a->va가 page_b->va보다 작지 않다
```

같은 주소여도 `<` 비교 결과는 `false`다.

## 왜 ==가 아니라 less를 두 번 쓰는가

Pintos hash table은 `equal` 함수를 따로 받지 않는다. 대신 `less` 함수만 받고, 같은 key인지 판단할 때 다음 조건을 사용한다.

```c
if (!h->less (hi, e, h->aux) && !h->less (e, hi, h->aux))
    return hi;
```

의미는 다음과 같다.

```text
hi < e 도 아니고
e < hi 도 아니면
hi와 e는 같은 key다
```

`page_less()`가 `va` 기준으로 비교한다면 위 조건은 결국 다음 의미가 된다.

```text
stored_page->va == key_page->va
```

여기서 `hi == e`처럼 포인터 비교를 쓰면 안 된다. `hi`와 `e`는 `struct hash_elem *`이므로, 포인터 비교는 "같은 객체 주소인가"만 본다. 하지만 SPT에서 필요한 것은 객체 주소가 아니라 "같은 가상 page 주소를 나타내는가"이다.

예를 들어 두 `struct page` 객체가 서로 다른 위치에 있어도, 둘 다 같은 `va`를 key로 가진다면 SPT에서는 중복 page로 봐야 한다.

```text
page1 객체 주소는 다름
page2 객체 주소도 다름

page1->va == page2->va
```

따라서 SPT hash table에서는 `page_hash()`와 `page_less()`가 모두 같은 기준인 `va`를 봐야 한다.

```text
page_hash() = va로 bucket 결정
page_less() = va로 같은 key인지 판단
```

## hash_entry와 NULL 처리

`hash_find()`가 반환하는 값은 `struct page *`가 아니라 `struct hash_elem *`다. 그래서 찾은 원소가 있을 때는 `hash_entry()`로 원래 `struct page *`를 복구해야 한다.

```c
return e != NULL ? hash_entry (e, struct page, hash_elem) : NULL;
```

이 코드는 풀어 쓰면 다음과 같다.

```c
if (e != NULL) {
    return hash_entry (e, struct page, hash_elem);
} else {
    return NULL;
}
```

`e == NULL`이면 찾은 원소가 없다는 뜻이다. 이때 바로 `hash_entry(e, ...)`를 호출하면 잘못된 주소 계산이 되므로, 먼저 `NULL`인지 확인해야 한다.

## spt_find_page에서 pg_round_down을 쓰는 이유

SPT의 key는 보통 page 시작 가상주소다.

```text
0x8048000 ~ 0x8048fff
```

이 범위에 속한 주소는 모두 같은 page를 의미한다. 그런데 page fault가 반드시 page 시작 주소에서 발생하는 것은 아니다.

```text
fault address = 0x8048123
page key      = 0x8048000
```

그래서 `spt_find_page()`에서는 입력 주소를 그대로 쓰면 안 되고, 먼저 page boundary로 내려야 한다.

```c
page.va = pg_round_down (va);
```

반대로 `pg_round_up()`을 쓰면 `0x8048123`이 `0x8049000`이 되어 다음 page를 찾게 된다. lookup에서는 "이 주소가 속한 page의 시작 주소"가 필요하므로 `pg_round_down()`이 맞다.

## pintos2 기준 구현 순서

`pintos2` 디렉터리 기준으로 보면 1번 작업은 `pg_round_down()`을 쓰기 위한 include 추가다.

```c
#include "threads/malloc.h"
#include "vm/vm.h"
#include "vm/inspect.h"
#include "threads/vaddr.h"
```

`pg_round_down()`은 `include/threads/vaddr.h`에 정의되어 있으므로, `vm.c`에서 쓰려면 이 헤더를 포함해야 한다.

2번 작업은 `spt_find_page()` 구현이다.

```c
struct page *
spt_find_page (struct supplemental_page_table *spt, void *va) {
    struct page page;
    struct hash_elem *e;

    page.va = pg_round_down (va);
    e = hash_find (&spt->pages, &page.hash_elem);

    return e != NULL ? hash_entry (e, struct page, hash_elem) : NULL;
}
```

이 함수에서 임시 `struct page page`는 실제로 SPT에 넣을 page가 아니다. 검색용 key 객체다. `page.va`만 채워서 `hash_find()`에 넘기면, PintOS hash table은 `page_hash()`와 `page_less()`를 이용해 같은 `va`를 가진 실제 page를 찾는다.

이제 `spt`, `va`를 실제로 사용하므로 함수 인자에 붙어 있던 `UNUSED`는 제거해야 한다.

## 오늘의 결론

SPT hash table에서 `page_less()`의 목적은 정렬 자체가 아니라, `va`를 key로 삼아 같은 page인지 판단하게 만드는 것이다. PintOS hash table은 `equal` 함수 없이 `less` 두 번으로 동등성을 판단하므로, `page_hash()`와 `page_less()`는 반드시 같은 기준인 page-aligned `va`를 사용해야 한다.

`spt_find_page()`는 이 원리를 직접 사용하는 첫 함수다. 입력 주소를 `pg_round_down()`으로 page 시작 주소에 맞추고, 임시 page key를 만들어 `hash_find()`로 찾은 뒤, 결과가 있으면 `hash_entry()`로 `struct page *`를 복구한다.
