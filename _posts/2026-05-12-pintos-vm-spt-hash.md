---
title: "PintOS VM: SPT와 hash table 골격 이해"
date: 2026-05-12 23:30:00 +0900
categories:
  - TIL
tags:
  - PintOS
  - OS
  - VirtualMemory
  - SPT
  - HashTable
  - C
  - Git
---

## 오늘 다룬 내용

오늘은 PintOS Project 3 VM을 구현하기 전에 SPT, page, frame, hash table의 관계를 정리했다. 처음에는 `writable` 같은 필드를 "직접 추가해야 한다"는 말이 무엇인지부터 확인했고, 이후 `vm_alloc_page_with_initializer()`, page type, bucket, `hash_entry()`, `hash_bytes()`까지 이어서 이해했다.

구현 관점에서는 Notion에 정리해 둔 "월요일 구현 시작을 위한 VM 개념학습" 기준을 다시 확인했다. 바로 frame allocation이나 page fault 전체 흐름으로 들어가는 것이 아니라, 먼저 다음 세 함수의 기반을 잡는 것이 출발점이라는 결론을 냈다.

```c
supplemental_page_table_init()
spt_find_page()
spt_insert_page()
```

마지막에는 `dev` 브랜치에 병합된 내용을 `kms` 브랜치로 다시 맞추는 Git 흐름도 확인했다. `kms`를 `dev`에 머지하면 `dev`만 앞으로 가고 `kms`는 그대로 남는다는 점을 커밋 그래프로 정리했다.

## SPT는 왜 필요한가

PML4는 현재 실제로 매핑된 가상주소와 물리 frame의 연결을 관리한다. 하지만 VM에서는 아직 물리 frame에 올라오지 않은 page도 유효할 수 있다. lazy loading 대상 page가 대표적이다.

아직 PML4에 매핑되지 않은 page는 PML4 PTE에 권한 정보를 저장할 수 없다. 그래서 커널은 SPT 안의 `struct page` metadata에 필요한 정보를 들고 있어야 한다.

예를 들어 `writable`은 다음 시점에 필요하다.

```c
pml4_set_page(thread_current()->pml4, page->va, frame->kva, page->writable);
```

즉 `writable`은 "현재 PML4에 이미 적혀 있는 값"이 아니라, 나중에 page fault가 발생해서 실제 mapping을 만들 때 써야 하는 SPT metadata다.

오늘 정리한 SPT의 역할은 다음 한 문장으로 요약할 수 있다.

```text
SPT는 현재 프로세스의 유저 가상 page 주소를 key로 해서 struct page metadata를 찾는 per-process table이다.
```

## page type 정리

page type은 hash table의 영역을 뜻하는 것이 아니라, hash table 안에 들어가는 `struct page`가 어떤 종류의 page인지 나타내는 metadata다.

오늘 구분한 세 가지는 다음과 같다.

- `VM_UNINIT`: 아직 실제 내용이 올라오지 않은 lazy page
- `VM_ANON`: 특정 파일에 직접 묶이지 않은 anonymous page
- `VM_FILE`: 파일을 backing store로 가지는 file-backed page

`VM_UNINIT`은 "나중에 접근하면 어떤 방식으로 초기화할지"만 SPT에 등록된 상태다. 첫 page fault가 발생하면 `uninit_initialize()`를 거쳐 실제 `VM_ANON` 또는 `VM_FILE` 성격으로 바뀐다.

`VM_ANON`은 stack, heap, zero page처럼 특정 파일에 직접 묶이지 않은 page다. 메모리에서 쫓겨나면 보통 swap 영역으로 간다.

`VM_FILE`은 `mmap()`처럼 파일과 연결된 page다. 이 page는 필요할 때 파일에서 읽어오고, dirty 상태라면 eviction이나 `munmap()` 때 다시 파일에 써야 한다.

## vm_alloc_page_with_initializer의 역할

`vm_alloc_page_with_initializer()`는 이름 때문에 물리 메모리까지 바로 할당하는 함수처럼 보일 수 있다. 하지만 핵심 역할은 "가상 page 하나를 SPT에 예약 등록하는 것"이다.

흐름은 다음처럼 이해했다.

```text
1. upage가 이미 SPT에 있는지 확인
2. struct page 할당
3. type에 맞는 initializer 선택
4. uninit_new()로 lazy page 상태를 만든다
5. writable 같은 metadata를 page에 저장
6. spt_insert_page()로 SPT에 등록
```

중요한 점은 이 함수가 frame을 할당하거나 PML4에 매핑하지 않는다는 것이다. 실제 frame 할당과 PML4 mapping은 이후 `vm_claim_page()`와 `vm_do_claim_page()` 쪽에서 일어난다.

따라서 오늘 구현 시작 범위는 자연스럽게 SPT hash table 골격이 된다. SPT에 넣고 찾는 동작이 먼저 있어야 lazy page 등록도 가능하다.

## PintOS hash table 이해

Pintos의 hash table은 배열과 리스트가 결합된 구조다.

```text
hash table
  buckets[0] -> list
  buckets[1] -> list
  buckets[2] -> list
  buckets[3] -> list
```

`struct hash` 안에는 bucket 배열을 가리키는 `buckets`가 있고, 각 bucket은 `struct list`다. 서로 다른 key가 같은 bucket으로 들어가는 collision이 생길 수 있으므로 bucket 내부는 linked list로 관리된다.

배열과 리스트의 차이도 다시 정리했다.

- 배열은 메모리에 연속으로 놓이고 index로 바로 접근할 수 있다.
- 리스트는 원소가 흩어져 있어도 되고 `prev`, `next` 포인터로 연결된다.
- hash table은 먼저 배열 index로 bucket을 좁히고, 그 bucket의 list에서 실제 원소를 찾는다.

Pintos hash가 처음 bucket을 4개로 시작하는 이유도 확인했다. 작은 크기로 시작해서 메모리를 아끼고, bucket 수를 2의 거듭제곱으로 유지하면 index 계산을 bit 연산으로 단순하게 할 수 있다.

```c
size_t bucket_idx = h->hash(e, h->aux) & (h->bucket_cnt - 1);
```

## hash_elem과 hash_entry

오늘 가장 많이 헷갈린 부분은 `hash_elem`과 `hash_entry()`였다.

Pintos hash table은 실제 `struct page` 전체를 복사해서 저장하지 않는다. 대신 `struct page` 안에 hash table 연결용 멤버를 직접 넣는다.

```c
struct page {
    void *va;
    bool writable;
    struct hash_elem hash_elem;
};
```

그리고 `struct hash_elem` 안에는 실제로 `struct list_elem`만 들어 있다.

```c
struct hash_elem {
    struct list_elem list_elem;
};
```

처음에는 "hash_elem 안에는 list_elem만 있는데 어떻게 다시 page를 꺼내지?"가 헷갈렸다. 핵심은 `hash_elem`이 `struct page` 안에 박혀 있다는 점이다.

```text
struct page 전체
+----------------+
| va             |
+----------------+
| writable       |
+----------------+
| hash_elem      |
|  +----------+  |
|  | list_elem|  |
|  | prev     |  |
|  | next     |  |
|  +----------+  |
+----------------+
```

hash table이 알고 있는 것은 `struct hash_elem *`뿐이다. 하지만 그 주소가 `struct page` 내부의 어느 위치인지 알면, `offsetof` 계산으로 `struct page`의 시작 주소를 역산할 수 있다.

```c
struct page *page = hash_entry(e, struct page, hash_elem);
```

`hash_entry()`가 반환하는 것은 시작주소 하나다. 시작주소부터 끝주소까지 범위를 반환하는 것이 아니다. 다만 반환 타입이 `struct page *`이기 때문에 컴파일러는 그 시작주소를 기준으로 `page->va`, `page->writable` 같은 필드 offset을 계산해서 접근할 수 있다.

## hash_bytes와 해시값

`hash_bytes()`도 헷갈렸던 지점이다.

```c
return hash_bytes(&page->va, sizeof page->va);
```

이 함수는 주소와 크기를 반환하는 함수가 아니다. 주소와 크기를 인자로 받아서, 그 메모리 영역의 바이트를 읽고 `uint64_t` 해시값 하나를 반환한다.

```text
입력:
- &page->va
- sizeof page->va

출력:
- uint64_t 해시값
```

해시값은 어떤 데이터를 빠르게 분류하기 위해 만든 숫자다. SPT에서는 `va` 값을 해시값으로 바꾸고, 그 해시값으로 bucket index를 정한다.

```text
va
-> hash_bytes()
-> hash value
-> bucket index
-> bucket 안의 list에서 page_less()로 실제 비교
```

이 구조 덕분에 모든 page를 처음부터 끝까지 순회하지 않고, 먼저 특정 bucket으로 탐색 범위를 줄일 수 있다.

## SPT hash table 구현 골격

SPT를 hash table로 구현하려면 `struct supplemental_page_table` 안에 `struct hash`를 둔다.

```c
struct supplemental_page_table {
    struct hash pages;
};
```

여기서 `pages`는 hash table을 가리키는 포인터가 아니라 hash table 자체다. 타입은 `struct hash`이고, 필드 이름이 `pages`다.

```text
struct supplemental_page_table
└── struct hash pages
    ├── elem_cnt
    ├── bucket_cnt
    ├── buckets
    ├── hash
    └── less
```

그래서 초기화할 때는 `pages` 필드의 주소를 넘겨야 한다.

```c
hash_init(&spt->pages, page_hash, page_less, NULL);
```

`&spt->pages`의 타입은 `struct hash *`이므로 `hash_init()`의 첫 번째 인자로 맞다.

또 `page_hash()`와 `page_less()`에서 `aux UNUSED`를 쓰는 이유도 정리했다. PintOS hash API의 함수 포인터 타입이 `void *aux` 인자를 요구하기 때문에 함수 선언에는 포함해야 한다. 하지만 SPT에서는 비교 기준이 `page->va`뿐이라 `aux`를 실제로 쓰지 않는다. 그래서 unused parameter warning을 막기 위해 `UNUSED`를 붙인다.

## 함수별 설명

`page_hash()`는 `struct page` 안의 `va`를 기준으로 해시값을 만든다. SPT에서 page를 구분하는 key가 page-aligned user virtual address이기 때문이다.

`page_less()`는 두 page의 `va`를 비교한다. PintOS hash table은 같은 원소인지 판단할 때 hash 함수와 less 함수를 함께 사용하므로, 둘 다 같은 기준인 `va`를 봐야 한다.

`supplemental_page_table_init()`은 현재 프로세스의 SPT 안에 있는 `pages` hash table을 사용 가능한 빈 table 상태로 만든다.

`spt_find_page()`는 입력 주소를 그대로 쓰면 안 된다. page fault 주소가 `0x401123`처럼 page 중간 주소일 수 있기 때문이다. 그래서 `pg_round_down()`으로 page 시작 주소인 `0x401000`으로 내린 뒤 찾아야 한다.

```text
spt_find_page(spt, va)
-> pg_round_down(va)
-> 임시 page key 생성
-> hash_find()
-> 찾으면 hash_entry()로 struct page * 반환
-> 없으면 NULL 반환
```

`spt_insert_page()`는 새 `struct page`를 SPT에 넣는다. PintOS의 `hash_insert()`는 삽입 성공 시 `NULL`을 반환하고, 같은 key가 이미 있으면 기존 element를 반환한다.

따라서 성공 조건은 다음과 같이 잡아야 한다.

```c
return hash_insert(&spt->pages, &page->hash_elem) == NULL;
```

이 조건 덕분에 같은 `va`를 가진 page가 하나의 SPT에 중복 등록되는 것을 막을 수 있다.

## frame과 file-backed page

SPT를 공부하면서 frame과 frame table의 차이도 같이 정리했다.

```text
frame table = 전체 물리 frame들을 관리하는 자료구조
frame       = 물리 메모리 4KB 한 칸을 나타내는 객체
```

SPT는 가상 page metadata를 관리하고, frame table은 실제 물리 frame들을 관리한다. page fault가 나면 SPT에서 `struct page`를 찾고, frame table에서 사용할 frame을 얻은 뒤, 둘을 연결하고 PML4 mapping을 만든다.

file-backed page에서는 파일 metadata를 함부로 버리면 안 된다. file-backed page의 backing store는 실제 파일이기 때문이다. page fault 때 다시 파일에서 읽어야 하고, dirty page라면 eviction이나 `munmap()` 때 파일에 다시 써야 한다.

단, dirty가 아닌 file-backed page의 물리 frame 내용은 버릴 수 있다. 나중에 다시 fault가 나면 파일에서 다시 읽으면 되기 때문이다. 버릴 수 있는 것은 frame의 메모리 내용이지, file, offset, read bytes 같은 metadata가 아니다.

## Git에서 머지 방향 이해

마지막으로 `kms`와 `dev` 브랜치 관계를 확인했다. `kms`를 `dev`에 머지했는데 왜 둘이 다르게 보이는지 헷갈렸다.

정리하면 머지는 방향이 있다.

```text
kms를 dev에 머지한다
-> dev가 kms 내용을 가져간다
-> kms가 자동으로 dev와 같아지는 것은 아니다
```

실제 그래프는 다음 구조였다.

```text
dffc60c  origin/dev
|\
| * 6dbc367  origin/kms
|/
a67b236
```

`dffc60c`는 GitHub에서 `kms`를 `dev`에 머지하면서 생긴 merge commit이고, `origin/kms`는 아직 이전 커밋인 `6dbc367`에 머물러 있었다. 그래서 로컬 `kms`에 `origin/dev`를 fast-forward로 당긴 뒤, 다시 `origin/kms`로 push해서 세 브랜치 포인터를 맞췄다.

```text
HEAD -> kms
origin/kms -> dffc60c
origin/dev -> dffc60c
```

## 헷갈렸던 점

처음에는 page type이 hash table의 영역처럼 느껴졌다. 하지만 page type은 hash table의 bucket이나 영역이 아니라, hash table에 들어가는 `struct page`의 속성이다. hash table은 `va -> struct page`를 찾기 위한 자료구조이고, page type은 찾은 page가 어떤 종류인지 설명하는 metadata다.

또 `hash_elem` 안에 `list_elem`만 있는데 어떻게 `struct page`를 얻는지 헷갈렸다. 답은 `hash_elem`이 `struct page` 안에 포함되어 있기 때문에, 그 멤버 주소에서 바깥 구조체의 시작주소를 역산할 수 있다는 것이다.

`list_elem`의 `prev`, `next`를 직접 골라서 써야 하는지도 헷갈렸다. 결론은 직접 건드리지 않는다. `hash_insert()`, `hash_find()`, `hash_delete()` 같은 PintOS hash/list 라이브러리가 내부에서 관리한다.

`hash_bytes()`도 주소와 size를 반환한다고 착각할 수 있었지만, 실제로는 주소와 size를 입력받아 해시값 하나를 반환하는 함수다.

## 오늘의 결론

오늘의 핵심은 PintOS VM 구현의 첫 단계를 SPT hash table 골격으로 좁힌 것이다.

SPT는 프로세스별로 존재하고, 그 안의 `pages` hash table은 page-aligned user virtual address를 key로 `struct page` metadata를 관리한다. `struct page`는 `writable`, `hash_elem`, page type 같은 정보를 담고, 실제 frame에 올라가기 전에도 lazy page의 상태를 설명한다.

구현 순서는 명확해졌다.

```text
1. struct page에 writable, hash_elem 추가
2. struct supplemental_page_table에 struct hash pages 추가
3. page_hash(), page_less() 작성
4. supplemental_page_table_init()에서 hash_init()
5. spt_find_page()에서 pg_round_down() 후 hash_find()
6. spt_insert_page()에서 hash_insert() 반환값으로 중복 삽입 방지
```

이 기반이 잡히면 다음 단계인 `vm_alloc_page_with_initializer()`에서 lazy page를 SPT에 등록하고, 이후 page fault와 claim 흐름으로 연결할 수 있다.
