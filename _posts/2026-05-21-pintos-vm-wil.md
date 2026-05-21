---
title: "5월 15일-21일 WIL: PintOS VM을 SPT에서 Lazy Loading 발표자료까지 연결하기"
date: 2026-05-21 14:41:32 +0900
categories:
  - WIL
tags:
  - Retrospective
  - Weekly
  - PintOS
  - OS
  - VirtualMemory
  - SPT
  - PageFault
  - LazyLoading
  - Security
---

## 이번 주에 다룬 내용

이번 주는 PintOS Project 3 VM을 "자료구조 하나를 채우는 일"이 아니라, page fault가 발생했을 때 운영체제가 어떤 정보를 근거로 복구하는지 연결해서 이해하는 데 집중했다. 시작점은 SPT였고, 중간에는 `vm_try_handle_fault()`, `vm_do_claim_page()`, frame allocation, `setup_stack()`을 지나 lazy loading 구현 흐름까지 이어졌다.

마지막에는 구현 내용을 발표자료로 정리하면서 "왜 VM을 써야 하는가"도 다시 설명했다. 단순히 메모리를 아끼기 위해서가 아니라, 가상 주소와 물리 frame을 분리하고, page fault 기반 lazy loading을 가능하게 하며, 프로세스와 커널의 메모리 경계를 보안적으로 지키기 위한 구조라는 점이 이번 주의 핵심이었다.

이번 주의 큰 흐름은 다음과 같다.

```text
SPT에 page metadata 등록
-> page fault 발생
-> fault address로 SPT 조회
-> vm_claim_page()
-> frame 확보
-> PML4에 virtual address와 frame mapping
-> swap_in 또는 lazy_load_segment()
```

## SPT는 page의 현재 상태를 설명한다

처음에는 SPT를 "page가 들어 있는 곳"처럼 생각하기 쉬웠다. 하지만 이번 주에 정리한 기준은 다르다. SPT에는 실제 데이터가 들어 있는 것이 아니라, 특정 virtual page를 어떻게 다뤄야 하는지에 대한 metadata가 들어 있다.

예를 들어 실행 파일의 segment를 lazy loading으로 등록할 때는 아직 RAM에 내용이 올라오지 않았을 수 있다. 그래도 SPT에는 해당 virtual address, writable 여부, page type, initializer, lazy loading에 필요한 auxiliary data가 남아 있다. 그래서 page fault가 났을 때 운영체제는 "이 접근이 합법적인가", "어디서 내용을 읽어와야 하는가", "어떤 frame에 올려야 하는가"를 판단할 수 있다.

`spt_insert_page()`를 볼 때 가장 중요했던 부분은 `hash_insert()`의 반환값이었다.

```c
return hash_insert (&spt->pages, &page->hash_elem) == NULL;
```

`hash_insert()`가 `NULL`을 반환한다는 것은 같은 key의 기존 원소가 없어서 새 원소 삽입에 성공했다는 뜻이다. SPT에서 key는 page boundary로 내린 virtual address이므로, 같은 가상 page가 중복 등록되는 것을 막을 수 있다.

## page fault는 항상 실패가 아니다

이번 주에 가장 많이 붙잡은 표현 중 하나가 "legal page fault"였다. 처음에는 page fault가 곧 잘못된 접근처럼 느껴졌지만, VM에서는 page fault가 정상적인 복구 흐름의 진입점이 될 수 있다.

예를 들어 아직 RAM에 올라오지 않은 실행 파일 segment에 접근하면 CPU는 page fault를 발생시킨다. 이때 fault address는 x86-64의 CR2 레지스터에 저장된다. 커널은 이 주소를 기준으로 SPT를 조회하고, 해당 page가 SPT에 등록되어 있으며 권한 조건을 만족하면 frame을 할당하고 page 내용을 채워 넣는다.

정리하면 page fault는 두 종류로 나눠서 봐야 한다.

- 잘못된 접근: null pointer, 권한 위반, 등록되지 않은 주소 접근
- 복구 가능한 접근: lazy loading 대상 page, stack growth 대상 page, swap out되었다가 다시 필요한 page

`vm_try_handle_fault()`는 이 둘을 가르는 함수다. `addr == NULL`이면 바로 실패해야 하고, write fault인데 page가 writable이 아니면 실패해야 한다. 반대로 SPT에 등록된 page이고 조건을 통과하면 `vm_claim_page()`로 넘겨 실제 frame과 연결한다.

## claim은 예약된 page를 실제 메모리에 올리는 단계다

`claim`이라는 단어도 이번 주에 따로 정리했다. SPT에 page metadata가 있다는 것은 "이 주소를 나중에 사용할 수 있다"는 예약에 가깝다. 실제로 사용할 수 있으려면 물리 frame을 확보하고, page table에 mapping하고, 필요한 내용을 채워야 한다.

이 흐름이 `vm_do_claim_page()`에 들어 있다.

```text
vm_do_claim_page(page)
-> vm_get_frame()
-> frame->page = page
-> page->frame = frame
-> pml4_set_page(thread_current()->pml4, page->va, frame->kva, page->writable)
-> swap_in(page, frame->kva)
```

여기서 `pml4_set_page()`는 단순히 비교만 하는 함수가 아니라 실제 mapping을 수행하고 성공 여부를 반환한다. 그래서 다음 코드는 "mapping을 시도했는데 실패하면"이라는 뜻이다.

```c
if (!pml4_set_page (...)) {
    ...
}
```

이 부분을 놓치면 `if` 조건문 안에서 왜 side effect가 생기는지 헷갈리기 쉽다.

## frame은 실제 page와 관리 구조를 나눠서 봐야 한다

`vm_get_frame()`을 보면서 `kva`와 `struct frame`의 차이도 정리했다. `palloc_get_page()`가 반환하는 `kva`는 실제로 사용할 한 page 크기의 메모리다. 반면 `struct frame`은 그 frame을 추적하기 위한 관리 구조체다.

따라서 다음 두 실패는 서로 다르다.

```text
kva == NULL
-> 실제 page 크기 메모리를 못 얻음

frame == NULL
-> frame 관리 구조체 malloc에 실패함
```

이미 `kva`를 얻은 뒤에 `frame` metadata 할당이 실패하면 `palloc_free_page(kva)`로 되돌려야 한다. mapping에 실패했을 때도 마찬가지다. 이번 주에는 "실제 메모리"와 "그 메모리를 관리하는 구조체"를 분리해서 보는 것이 frame 코드를 읽는 기준이 되었다.

## setup_stack은 VM 경로에서 봐야 한다

`setup_stack()`도 한 번 헷갈렸던 지점이다. PintOS skeleton에는 VM을 쓰지 않는 코드 경로와 VM을 쓰는 코드 경로가 함께 있다. Project 3에서는 `#else /* VM */` 쪽을 봐야 한다.

비 VM 코드에서는 `palloc_get_page()`로 직접 page를 할당하고 `install_page()`로 매핑한다. 하지만 VM 코드에서는 stack page도 SPT에 등록하고 claim하는 흐름으로 가야 한다.

핵심은 다음과 같다.

```text
stack_bottom = USER_STACK - PGSIZE
-> vm_alloc_page(VM_ANON | VM_MARKER_0, stack_bottom, true)
-> vm_claim_page(stack_bottom)
-> rsp = USER_STACK
```

그래서 VM 버전의 `setup_stack()`에서는 `uint8_t *kpage`를 직접 들고 있을 필요가 없다. 직접 물리 page를 할당하는 것이 아니라 VM 계층을 통해 page를 등록하고 claim해야 하기 때문이다.

## lazy loading은 파일 읽기를 접근 시점으로 미룬다

이번 주 후반에는 lazy loading 구현을 Notion에 먼저 정리한 뒤 코드 흐름을 잡았다. 핵심은 `load_segment()`가 실행 파일 내용을 즉시 모두 읽어오는 대신, page마다 필요한 정보를 `aux`에 담아 SPT에 등록해 둔다는 것이다.

`lazy_load_segment_aux`에는 보통 다음 정보가 들어간다.

```text
file
ofs
read_bytes
zero_bytes
```

이 정보는 page fault가 발생해 실제 frame이 확보된 뒤 `lazy_load_segment()`에서 사용된다. 해당 page에 필요한 만큼 파일에서 읽고, 남은 공간은 0으로 채운다.

중요한 검사는 다음 형태였다.

```c
if (file_read (file, page->frame->kva, page_read_bytes) != (int) page_read_bytes)
    return false;
```

왼쪽은 실제로 읽은 byte 수이고, 오른쪽은 기대한 byte 수다. 같지 않으면 파일에서 page 내용을 정상적으로 가져오지 못한 것이므로 실패해야 한다. 이후 `memset()`으로 `zero_bytes`만큼 0을 채우면 lazy loaded page가 완성된다.

## VM을 쓰는 이유는 성능만이 아니다

발표자료를 준비하면서 "물리 메모리를 그냥 직접 쓰면 안 되는가"라는 질문을 다시 정리했다. 결론은 직접 쓸 수는 있어도 운영체제가 제공해야 하는 격리, 복구, 확장 기능이 크게 무너진다는 것이다.

VM이 해 주는 일은 다음처럼 요약할 수 있다.

```text
virtual address
-> page
-> frame
-> physical memory
```

이 분리가 있기 때문에 프로세스는 자신만의 연속적인 주소 공간을 가진 것처럼 실행될 수 있다. 실제 물리 frame은 커널이 필요에 따라 배치하고 교체한다. 이 구조 위에서 lazy loading, stack growth, `mmap()`, swap, eviction이 가능해진다.

이번에 추가로 정리한 보안 측면도 중요하다. VM은 단순한 편의 기능이 아니라 메모리 보호 경계다.

- 유저 프로세스가 커널 메모리를 직접 읽거나 쓰지 못하게 한다.
- 한 프로세스가 다른 프로세스의 주소 공간을 침범하지 못하게 한다.
- writable 여부와 user/kernel 권한을 page 단위로 검사할 수 있게 한다.
- 잘못된 포인터 접근을 page fault로 잡아 프로세스를 종료하거나 복구 흐름으로 보낼 수 있다.

여기서 보안은 암호화만을 뜻하지 않는다. 기밀성과 무결성을 깨는 잘못된 메모리 접근을 막는 것도 운영체제 보안의 핵심이다. VM이 없다면 프로세스 격리가 약해지고, 악의적이거나 버그가 있는 프로그램이 다른 주소 공간을 훼손할 가능성이 커진다.

## 이번 주 결과물

이번 주에는 구현 이해와 문서화가 같이 진행되었다.

- `spt_insert_page()`, `vm_try_handle_fault()`, `vm_do_claim_page()` 흐름 정리
- `vm_get_frame()`에서 `kva`와 `struct frame`의 역할 분리
- VM 버전 `setup_stack()` 구현 위치와 흐름 정리
- lazy loading 구현 초안과 `lazy_load_segment_aux` 설명 정리
- `anon_swap_in()`과 `anon_swap_out()`의 현재 구현 범위 확인
- Notion에 `setup_stack`, lazy loading, VM 사용 이유 문서 작성
- Canva와 HTML 발표자료에 VM 역할, 직접 물리 메모리 사용의 문제, 보안 이유 반영

구현 검증에서는 macOS ARM 환경과 PintOS x86 빌드 옵션 차이 때문에 전체 `make check`까지 안정적으로 확인하지는 못했다. 그래서 발표자료의 테스트 상태에도 통과 결과를 임의로 쓰지 않고, 전체 suite 검증이 아직 완료되지 않았다는 점을 명시했다. 구현 기록에서도 "확인한 것"과 "아직 확인하지 못한 것"을 구분하는 것이 중요했다.

## 아쉬운 점

개념을 하나씩 이해하는 데는 도움이 되었지만, 중간에 VM을 쓰지 않는 코드 경로와 VM 코드 경로를 섞어 보면서 시간이 조금 걸렸다. 특히 `setup_stack()`처럼 `#ifdef VM`으로 갈라지는 함수는 현재 프로젝트 단계에서 어떤 경로가 실제 대상인지 먼저 확인해야 한다.

또 lazy loading을 코드에 넣기 전에 `aux` 구조와 ownership을 더 빨리 확정했으면 좋았을 것 같다. file pointer, offset, read bytes, zero bytes는 page fault 시점까지 살아 있어야 하므로 단순히 지역 변수로 넘기는 구조가 아니라 heap에 저장하고 해제 시점까지 생각해야 한다.

테스트도 다음 단계에서는 더 명확히 잡아야 한다. 전체 PintOS 빌드가 로컬 환경에서 막히면 최소한 개별 object compile, 문법 검사, target 환경에서의 `make check`를 분리해서 기록해야 한다. 그래야 발표자료나 회고에서 구현 상태를 과장하지 않을 수 있다.

## 이번 주 한 줄 회고

이번 주의 핵심은 SPT를 출발점으로 page fault, claim, frame mapping, lazy loading을 하나의 복구 흐름으로 연결하고, 그 구조가 메모리 절약뿐 아니라 프로세스 격리와 보안을 위해서도 필요하다는 점을 이해한 것이다.
