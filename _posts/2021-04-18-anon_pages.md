---
layout: post
title: "About anonymous page"
tags: [linux, kernel, Memory]
---
Reference: https://www.kernel.org/doc/html/latest/admin-guide/mm/concepts.html

# Page의 종류
페이지는 크게 두 분류로 나뉜다.
1. 익명 페이지(`Anonymous page`)
2. 파일-기반 페이지(`File-backed page`)[^1]

파일-기반 페이지는 **파일으로부터 매핑된 페이지**를 뜻한다.  
익명 페이지는 **파일으로부터 매핑되지 않은, 커널로부터 할당된 페이지**를 뜻한다.  
본 글에서는 `file-backed page`는 다루지 않는다.

## Anonymous Page
익명 페이지는 커널로부터 프로세스에게 할당된 일반적인 메모리 페이지이다.  
즉, **익명 페이지는 힙을 거치지 않고 할당받은 메모리 공간**이다.  
(힙도 익명 페이지이다. malloc, new 같은 메모리 할당자는 익명 페이지에서 일부 메모리를 잘라 할당 받는것이다.)  
먼저 '익명' 이라는 뜻은 파일에 기반하고 있지 않은(파일로부터 매핑되지 않은) 페이지라는 뜻이다.  
페이지가 파일에 매핑되어 있다면, 그 메모리는 파일 내용을 담고 있을 것이다.  
하지만 익명 페이지는 파일에 매핑되어 있지 않았기 때문에 0으로 초기화된 값을 담고 있다.

프로세스가 `mmap()`으로 커널에게 익명 페이지를 할당 요청하게 되면, 커널은 프로세스에게 가상 메모리 주소 공간을 부여하게 된다.  
부여된 가상 메모리 공간은 아직까지는 실제 물리 메모리 페이지로 할당되지 않은 공간이다.  
부여된 가상 메모리는 메모리 읽기 쓰기시, 다음과 같은 커널 도움을 받아 zero 페이지로 에뮬레이션 되거나, 실제 물리 페이지로 매핑된다.
1. 프로세스가 그 메모리 공간에 읽기 작업 시, 커널은 zero로 초기화된 메모리 페이지 (file-backed page with `/dev/zero`)을 제공한다.
2. 프로세스가 그 메모리 공간에 쓰기 작업 시, 커널은 실제 물리 페이지를 할당하고 write된 데이터를 보관한다.

익명 페이지는 `private` 또는 `shared`로 할당받을 수 있다.  
프로세스의 힙과 스택이 `private`로 할당된 `anonymous page`이다.  
`shared`는 프로세스간 통신을 위해 사용되는 `anonymous page`이다.  

익명 페이지를 할당 받으려면 다음과 같은 코드가 이용된다.
```C
void* ptr = mmap(NULL, size, PROT_READ | PROT_WRITE, MAP_ANONYMOUS|MAP_SHARED, -1, 0);
// anonymous page는 fd가 -1이여야 하며, offset은 0이여야 한다.
// 이후 익명 페이지를 읽으면 0으로 초기화된 값이 읽히며,
// 익명 페이지를 쓰면 실제 물리 메모리 페이지가 할당된다.
```
## Reverse Mapping of Anonymous page
**To be moved to reverse mapping section**  
시스템에서 address는 virtual address에서 physical address로 변환된다.  
변환되는 과정은 page table을 통해 이루어진다.  
그러면 physical address에서 virtual address로 변환해야 하는 경우 어떨까?  
그런 일이 필요할까?  
우리는 physical address에서 virtual address로 주소를 변환하는 과정을 reverse mapping이라 부른다.  
(정확히는 page table의 PTE(virtual address)를 알아내는 작업이다.)

### 왜 reverse mapping이 필요한가?
대표적으로 page reclaiming이 있다.  
어떤 물리 페이지를 swapout하고 싶다면, 관련된 모든 virtual address(PTE)를 찾아 invalidate시켜 주어야 한다.
이 때 reverse mapping이 필요하다.

### 초기의 reverse mapping
단순히 `struct page`에 각각의 PTE를 linked list 형태로 가지고 있어, 바로 pte에 접근할 수 있었다.  
하지만 process 개수가 증가할수록 `struct page`가 가진 PTE의 수가 선형적으로 증가한다는 문제가 있었다.  
file-backed page에서 shared library의 경우 전체 프로세스의 개수만큼 PTE 정보를 유지해야 했다.  
실제 kernel 2.6에서는 이러한 방식으로 관리가 되었고, 메모리 낭비가 상당히 심했다.  

### Reverse mapping key idea
결국 `struct page`가 linked list의 형태로 PTE를 가지고 있기 때문에 메모리 낭비가 심했다.  
메모리 낭비를 줄이기 위해 **object-based reverse mapping**이 등장하였다.  
Revserse mapping의 목적은 physical address로부터 PTE를 얻어내는 것이다.  
만약 physical address로부터 virtual address를 얻어낸다면, page table을 통해 PTE를 쉽게 얻어낼 수 있다.  
이 때문에, `struct page`가 매핑된 `vm_area_struct`[^2]가 있다면, virtual address를 다음과 같이 계산할 수 있지 않을까?
```C
unsigned long vaddr = vm_area_struct->vm_start + page_offset
```
`page_offset`은 virtual address에 매핑할 때, `page->index`에 미리 저장해 두었다고 가정하자.  
결국, `vm_area_struct`만 얻어내면 된다. `struct page`가 `vm_area_struct`만 들고 있으면 해결되는 문제이다.  
문제는 `vm_area_struct`는 프로세스의 개수, 그리고 프로세스 내 매핑에 따라 여러개가 존재한다.   
여러 `vm_area_struct`가 존재하므로 이를 묶어서 관리해야 한다.    
묶어서 관리하는 구조체를 `anon_vma`라 하자.  
![](https://user-images.githubusercontent.com/41955137/115137872-8c3dd100-a063-11eb-842a-fa29c89c3933.png)[^3]  

다음과 같이 `anon_vma`가 여러 `vm_area_struct`를 들고 있고, 이를 이용해 virtual address를 구한다.  
virtual address를 `mm_struct`에 있는 page table을 이용하여 PTE를 구하게 된다.  

여기서 중요한 점은 `page->index`가 한개(위 그림에서 `page descr`)임을 상기하자.  
즉, `page_offset`은 다른 프로세스의 가상주소 공간 내에서도 바뀌지 않음을 의미한다.  
심지어 `mremap()`을 통해 다른 virtual address로 매핑시킨다 하더라도, `page_offset`은 변하지 않는다.

>**mremap - remap a virtual memory address**  
> ...  
>(offsets relative to the starting address of the mapping  
>should be employed).

이런식으로 physical address로부터 PTE를 구할 수 있고,  
문제가 되었던 메모리 낭비 문제는 프로세스마다 가지고 있는 `vm_area_struct`를 재활용함으로써 해결할 수 있었다.  
만약 프로세스가 하나 더 생기면 그 프로세스의 `vm_area_struct`를 `anon_vma` 안에 넣어주면 끝이다.  
다만, 이 방식은 `struct page`에서 `PTE`를 얻어내는 방식에 비해 약간 느릴 수 있다.  
1. 기존 방식: struct page -> PTE  
2. 새로운 방식: struct page -> anon_vma -> vm_area_struct -> mm_struct -> page table -> PTE  
추가적인 오버헤드가 생겼으나, 그 오버헤드는 negligible하다.

---
**Reference**
1. [Yizhou Shan's Home Page](http://lastweek.io/notes/rmap/)
2. [Columbia W4118 by Jungfeng Yang](https://www.cs.columbia.edu/~junfeng/13fa-w4118/lectures/l20-adv-mm.pdf)
2. McCracken, Dave. "Object-based reverse mapping." Linux Symposium. 2004.
4. [The object-based reverse-mapping VM by corbet](https://lwn.net/Articles/23732/)

---

[^1]: 파일 기반 페이지라는 한글 번역은 구글 번역기에서 내놓은 번역이다. 그 외의 번역이 있는지 모르겠다.
[^2]: `vm_area_struct`는 프로세스 가상 메모리 공간에서 연속된 가상 메모리 공간을 관리하는 구조체이다.
[^3]: [Yizhou Shan's Home Page](http://lastweek.io/notes/rmap/)