---
title: "PintOS VM: SPT 삽입과 page fault 처리 흐름 잡기"
date: 2026-05-14 23:50:00 +0900
categories:
  - TIL
tags:
  - PintOS
  - OS
  - VirtualMemory
  - SPT
  - PageFault
  - C
  - Git
---

## 오늘 다룬 내용

오늘은 PintOS Project 3 VM 구현을 top-down으로 보기 시작했다. 전날까지는 SPT hash table에서 `page_less()`, `hash_find()`, `spt_find_page()`가 어떻게 맞물리는지 이해하는 데 집중했다. 오늘은 그 다음 단계로, 현재 코드에서 무엇이 구현되어 있고 무엇이 아직 남아 있는지 확인한 뒤 `spt_insert_page()`와 `vm_try_handle_fault()` 흐름을 잡았다.

핵심 흐름은 다음과 같다.

```text
page fault 발생
-> vm_try_handle_fault()
-> spt_find_page()
-> 권한 검사
-> vm_claim_page() 또는 vm_do_claim_page()
-> frame 할당
-> PML4 mapping
-> swap_in 또는 lazy load
```

아직 모든 VM 기능을 구현하는 단계는 아니다. 오늘의 기준은 Notion의 "월요일 구현 시작을 위한 VM 개념학습" 계획에 맞춰 SPT 골격을 먼저 완성하고, 그다음 page fault 처리 흐름으로 넘어가는 것이었다.

## 구현 상태 확인

처음 확인한 요구사항은 다음 세 함수였다.

```c
supplemental_page_table_init()
spt_find_page()
spt_insert_page()
```

`pintos` 기준으로는 일부 구조가 들어가 있었지만, 다음 문제가 남아 있었다.

- `struct supplemental_page_table` 안에 `struct hash pages`는 있음
- `supplemental_page_table_init()`에서 `hash_init()`은 호출하고 있음
- `spt_find_page()`는 TODO 상태였음
- `spt_insert_page()`도 TODO 상태였음
- `page_hash()`와 `page_less()`가 중복 정의된 상태였음
- `pintos/include/vm/vm.h`에는 병합 충돌 흔적이 남아 있었음

반면 `pintos2` 기준으로는 상태가 조금 달랐다.

- `struct page`에 `writable`, `hash_elem` 있음
- `struct supplemental_page_table`에 `struct hash pages` 있음
- `page_hash()`와 `page_less()` 있음
- `supplemental_page_table_init()` 구현되어 있음

그래서 `pintos2`에서 바로 해야 할 일은 `spt_find_page()`와 `spt_insert_page()`였다. frame allocation, lazy loading, page fault 전체 흐름은 그다음 단계로 보는 것이 계획과 맞았다.

## VM 타입의 의미

`load_segment()` 쪽에서 다음 코드처럼 `VM_ANON`을 넘기는 부분이 있었다.

```c
void *aux = NULL;
if (!vm_alloc_page_with_initializer (VM_ANON, upage,
            writable, lazy_load_segment, aux))
    return false;
```

여기서 중요한 점은 `VM_ANON`을 넘긴다고 해서 즉시 anonymous page가 만들어지는 것은 아니라는 것이다. `vm_alloc_page_with_initializer()`의 의도는 우선 `VM_UNINIT` page를 SPT에 등록하고, 나중에 page fault가 났을 때 저장해 둔 타입에 맞는 initializer를 실행하는 것이다.

흐름은 다음처럼 이해했다.

```text
vm_alloc_page_with_initializer()
-> type에 맞는 initializer 선택
-> uninit_new()
-> SPT에 VM_UNINIT page 등록
-> page fault 발생
-> uninit_initialize()
-> anon_initializer() 또는 file_backed_initializer()
-> lazy_load_segment()
```

`VM_ANON`은 파일에서 초기 내용만 읽어온 뒤에는 anonymous page처럼 swap 대상이 되는 실행 파일 segment에 사용할 수 있다. 반면 `VM_FILE`은 `mmap()`처럼 파일과 계속 연결되어 dirty page를 파일에 다시 반영해야 하는 경우에 쓴다.

즉 `type`은 `lazy_load_segment()` 자체를 고르는 값이 아니라, page fault 이후 이 page가 어떤 VM subtype으로 살아갈지를 정하는 값이다.

## spt_find_page 반영

`pintos2`의 `spt_find_page()` 구현 의도를 `pintos` 쪽에도 반영했다.

핵심은 입력 주소를 그대로 찾지 않고 page boundary로 내린 뒤 찾는 것이다.

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

처음에 헷갈린 부분은 `struct page *page`와 `struct page page`의 차이였다.

```c
struct page page;   /* 검색용 임시 객체 */
struct page *page;  /* 아직 아무 page도 가리키지 않는 포인터 */
```

`spt_find_page()`에서는 실제 page를 새로 할당하는 것이 아니라, `hash_find()`에 넘길 검색용 key 객체가 필요하다. 그래서 `struct page page;`가 맞다. `struct page *page;`로 선언하면 포인터가 어떤 유효한 객체도 가리키지 않은 상태에서 `page.va`에 접근하려 하게 된다. 게다가 포인터라면 필드 접근도 `page->va`가 되어야 한다.

이 변경은 `kms` 브랜치에 커밋하고 `origin/kms`로 push했다.

```text
872e55d Implement SPT page lookup
```

## spt_insert_page 방향

`spt_find_page()` 다음 순서는 `spt_insert_page()`다. 이 함수는 SPT hash table에 `struct page` metadata를 `va` 기준으로 등록한다.

기본 구현 방향은 다음과 같다.

```c
bool
spt_insert_page (struct supplemental_page_table *spt, struct page *page) {
    return hash_insert (&spt->pages, &page->hash_elem) == NULL;
}
```

`hash_insert()`의 반환값이 중요하다.

```text
NULL 반환      = 새 원소 삽입 성공
NULL이 아님    = 같은 key가 이미 있어서 기존 elem 반환
```

따라서 `hash_insert(...) == NULL`은 "중복 없이 삽입에 성공했는가"를 의미한다. SPT에서 key는 `page->va`이고, `page_hash()`와 `page_less()`가 `va` 기준으로 동작하므로 같은 가상주소 page가 중복 등록되는 것을 막을 수 있다.

기존 skeleton에는 다음 코드가 있었다.

```c
int succ = false;
```

이 변수는 유지해도 되지만 필수는 아니다. 바로 반환하면 더 단순하다.

```c
return hash_insert (&spt->pages, &page->hash_elem) == NULL;
```

다만 PintOS 스타일과 함수 반환 타입을 생각하면 변수를 둔다면 `int`보다 `bool`이 더 자연스럽다.

## vm_try_handle_fault 흐름

이제 SPT lookup과 insert 방향이 잡혔으므로 다음 top-down 학습 지점은 `vm_try_handle_fault()`다.

이 함수의 핵심 역할은 page fault가 VM에서 처리 가능한 fault인지 판별하고, 처리 가능하다면 해당 page를 claim하는 것이다.

흐름은 다음과 같이 잡았다.

```c
bool
vm_try_handle_fault (struct intr_frame *f UNUSED, void *addr,
        bool user UNUSED, bool write, bool not_present) {
    struct supplemental_page_table *spt = &thread_current ()->spt;
    struct page *page = NULL;

    if (addr == NULL)
        return false;

    if (is_kernel_vaddr (addr))
        return false;

    if (!not_present)
        return false;

    page = spt_find_page (spt, addr);
    if (page == NULL)
        return false;

    if (write && !page->writable)
        return false;

    return vm_do_claim_page (page);
}
```

각 검사의 의미는 다음과 같다.

- `addr == NULL`: null pointer fault는 처리하지 않는다.
- `is_kernel_vaddr(addr)`: 유저 프로세스가 커널 주소에 접근한 경우는 처리하지 않는다.
- `!not_present`: present bit가 없는 fault가 아니라 protection fault라면 지금 단계에서는 처리하지 않는다.
- `spt_find_page(spt, addr)`: SPT에 등록된 lazy page인지 확인한다.
- `write && !page->writable`: 쓰기 fault인데 read-only page면 실패해야 한다.
- `vm_do_claim_page(page)`: 실제 frame 할당, mapping, lazy load 쪽으로 내려간다.

여기서 `vm_claim_page(page->va)`를 호출할 수도 있지만, 이미 `spt_find_page()`로 `page`를 찾은 상태라면 `vm_do_claim_page(page)`가 더 직접적이다. 다시 `vm_claim_page()`로 가면 SPT lookup을 한 번 더 하게 된다.

## 다음 구현 흐름

top-down으로 보면 순서는 다음과 같다.

```text
1. vm_try_handle_fault()
2. vm_claim_page()
3. vm_do_claim_page()
4. vm_get_frame()
5. uninit_initialize()
6. lazy_load_segment()
```

다만 `vm_try_handle_fault()`의 `write && !page->writable` 검사가 의미 있으려면, 앞단의 `vm_alloc_page_with_initializer()`에서 반드시 `page->writable = writable;`이 저장되어야 한다.

오늘의 결론은 SPT가 단순 자료구조가 아니라 page fault 처리의 첫 관문이라는 점이다. fault 주소가 들어오면 SPT에서 해당 `struct page`를 찾고, 권한을 확인한 다음, 실제 frame claim과 lazy loading 흐름으로 이어진다. 따라서 `spt_find_page()`와 `spt_insert_page()`가 안정적으로 맞아야 `vm_try_handle_fault()` 이후 구현도 의미 있게 진행된다.
