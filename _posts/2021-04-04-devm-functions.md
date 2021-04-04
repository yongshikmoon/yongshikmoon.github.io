---
layout: post
title: "About devm"
tags: [linux, kernel, API, Resource]
---
# Device Resource Management

***

## devm이란?
리눅스 커널 디바이스 드라이버는 시스템 stability에 영향을 크게 끼친다.  
이 때문에, 리눅스 커널 모듈은 자신이 할당한 리소스에 대하여 세밀한 관리가 필요하다.  
하지만 가끔 디바이스 드라이버를 작성할때, 리소스에 대한 관리가 잘 되지 않을 때가 있다.  
(예를 들어, irq handler를 등록한 후, irq handler를 리소스 해제하지 않는 행위 등을 뜻한다.)  
일반 응용 개발과 달리, 커널 리소스가 제대로 관리되지 않은 경우, 시스템 크래시를 유발할 수 있다.  
<br>
이를 위해 리눅스 커널은 `devm_*` 이라는 함수를 가지고 있다.   
이 함수는 `struct device`가 할당 리소스의 생명 주기를 모두 관리할 수 있게 해주며,   
`struct device`가 해제될 시, 관리되고 있는 모든 리소스가 해제된다.   
(임의의 리소스를 해제할 수 있는 함수 또한 제공한다.)  
<br>
`devm_*` 함수로 할당된 모든 리소스는 `devres`라는 용어로 통칭된다.

## devm_* 함수 리스트
### devm_* 함수들
1. `devm_request_irq()`
2. `devm_kmalloc()`
3. `devm_get_free_pages()`
4. `__devm_alloc_percpu()`
5. `devm_***` 등[^1]   

위 함수들은 앞 인자에 `struct device* dev`가 있다는것이 다를 뿐, 그 이외에는 같다.

### devm_* 함수들의 동작 원리
#### 할당
할당은 다음과 같은 컨셉으로 이루어진다.
```cpp
{
    struct custom_devres* customdevres = devres_alloc(devm_xx_release, sizeof(custom_devres), GFP_KERNEL);
    // custom setting to custom_devres
    customdevres.order = a
    //
    devres_add(dev, customdevres);
}
```
위에서 `devres_alloc()`을 부르게 되면, 다음과 같이 메모리를 할당하게 된다.
1. `sizeof(devres) + sizeof(custom_devres)` 만큼을 커널 메모리(kmem_cache)에서 할당한다.
2. 만약 할당된 주소를 `X`라고 하면, `customdevres`에는 `X + sizeof(devres)`를 리턴하게 된다.

메모리 레이아웃은 다음과 같다.
```cpp
struct devres {
    struct devres_node node;
    u8 __aligned(ARCH_KMALLOC_MINALIGN) data[]; --> struct custom_devres;
};
```
union을 활용하면 다음과 같은 의미를 같는다.
```cpp
struct devres {
    struct devres_node node; // linked list head를 갖고 있는 부분
    union {
        u8 data[];
        struct custom_devres customdevres;
    }
};
```

다만 리눅스 커널은 `union` 형태처럼 정적 형태로 자료구조를 선언할 수 없기 때문에 `u8 data[];` 와 같이 변수를 선언한다. [^2]  
`devres_add()`함수를 통해 `customdevres`를 `dev.devres_head`에 링크드 리스트로 연결한다.  
`customdevres`는 `list_head`를 갖고 있지 않으나, `devres.node.entry`가 `list_head` 형태이므로 링크드 리스트로 연결할 수 있다.[^3]  
#### 해제
`devres_release_all()`은 모든 리소스와 `struct devres`를 해제한다.   
`devres_release_all()`은 `dev->devres_head`에 링크되어 있는 모든 `struct devres`를 커스텀 리소스 해제 함수를 부른 뒤,   `sturct devres`를 해제한다.  
`devres_remove()`는 이와 다르게 리소스를 해제하지 않고, `struct devres`만을 해제한다.  

### devres group
`devres`를 grouping해서 관리할 수 있는 기능이다.  
`devres_open_group()`을 통해 이후 할당되는 `devres`(by `devm`)은 opened group에 묶이게 된다.  
그리고 `devres_close_group()`을 통해 group을 닫을 수 있다.  
그룹 id에 어떠한 변수라도 넣을 수 있으며, `NULL`인경우 자동적으로 최근 open된 group이 선택된다.  
<br>
재미있는것은 `devres_open_group()`을 콜 하고 이후 `devres`관련된 함수 콜을 진행한 뒤, `devres_close_group()`으로 그룹을 닫으면, 다음과 같은 링크드 리스트가 `dev.devres_head`에 생성된다.  
```
<: open group (devres_group)
kmalloc: devres
iomap_resource: devres
>: closed group (devres_group)
```
이후 `dev_release_group()`을 호출하게 되면, group id에 해당하는 `<, >`을 찾고, 그 안에 있는 `devres`들을 해제하게 된다.  
만약 어떠한 에러가 발생하여 `>`가 발견되지 않으면, `<`로 시작하는 모든 `devres`를 할당 해제하게 된다.  
```c
  if (!devres_open_group(dev, NULL, GFP_KERNEL))
	return -ENOMEM;

  acquire A;
  if (failed)
	goto err;

  acquire B;
  if (failed)
	goto err;
  ...
  devres_close_group(dev, NULL);
  return 0;

 err:
  devres_release_group(dev, NULL);
  return err_code;
```
[^4]

***

[^1]: 풀 리스트는 [Kernel Document, devres.txt](https://www.kernel.org/doc/Documentation/driver-model/devres.txt)에서 확인할 수 있다.
[^2]: Zero sized array라고 불리는 기법이며, [GCC manual: Array of Length Zero](https://gcc.gnu.org/onlinedocs/gcc/Zero-Length.html)에서 확인할 수 있다. void*도 좋은 방법이나, 포인터 사이즈를 낭비하게 된다.
[^3]: `container_of()`를 통해 변수 `data`로부터 `struct devres` 주소를 구한다. 그 뒤, `node->entry`를 `dev.devres_head`에 링크드 리스트로 엮는다.
[^4]: [예문 코드](https://www.kernel.org/doc/Documentation/driver-model/devres.txt)