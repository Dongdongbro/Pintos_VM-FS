### 용어 정리

> 페이지 : 가상메모리의 연속적인 구역
> 

> 프레임 : 물리메모리의 연속적인 구역
> 

> 페이지 테이블 : 가상주소를 물리메모리로 전환시켜주기 위한 데이터 스트럭쳐
> 

> Eviction : 페이지를 프레임으로부터 제거하고 잠정적으로 스왑테이블 혹은 파일시스템에 올린다.
> 

> 스왑 테이블 : evicted 페이지들이 스왑 파티션에 올려진 정보들을 가진다.
> 

### 디자인 해야할 데이터 스트럭쳐

1. Supplemental page table : 프로세스 당 각 페이지를 위한 supplemental 데이터들을 추적할 자료 구조가 있어야 한다. 예를 들어 데이터의 위치(프레임/디스크/스왑파티션), 상응하는 커널 가상 주소 포인터, 유효한지 안한지 등등
    
    용도 2가지
    
    1) swap한 데이터 쓰고 싶을 때 → page fault 남 → supplemental page table에서 fault된 가상 페이지 찾음
    2) 페이지가 종료될 때, 자원 할당을 해제하기 위해 커널은 supplemental page table을 설계
    
    - demand paging에 의해 당장 필요한 부분은 Physical memory에 올라가 있고,
        
        그렇지 않은 건 swap 영역(disk)에 올라가 있는 그림
        
        ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/96b3efe1-a6c7-4a50-a276-c488785b0546/Untitled.png)
        
2. Frame table : 할당/삭제된 물리메모리 프레임 정보를 저장하는 전역 데이터 스트럭쳐
3. Swap table : 스왑 슬롯의 정보를 담는다.
    - [Swap area, page slot 에 대한 그림](https://m.blog.naver.com/PostView.naver?isHttpsRedirect=true&blogId=jevida&logNo=140194288224)
        
        ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/0d215c64-00e1-444d-8a42-72f2e642c3dc/Untitled.png)
        
4. File mapping table : 어떤 페이지에 어떤 메모리-mapped 파일들이 있는지 저장


### lazy loading의 개념

가상주소가 생성되었을 때, 핀토스는 가상주소에 페이지 구조체를 할당한다.

각 가상주소가 다른 목적으로 사용된다. (anonymous memory, file-backed memory)

- 페이지 구조체는 정보를 반영한다(페이지를 점검)
- 페이지 구조체 할당은 물리메모리 프레임이 가상주소에 할당되는 것을 의미하지는 않는다.

### 페이지 구조체와 페이지 운용

어떻게 페이지가 할당된 다음 프레임 정보들이 채워질까?

- 페이지는 uninit_page에 의해서 시작된다.
- 만약 페이지가 anonymous memory을 위한 것이라면 → anon_initializer
- 만약 페이지가 filed-backed memory을 위한 것이라면 → file_map_initializer

### Supplemental Page Table

- 각 페이지에 대한 추가 정보를 담고 있는 Supplements 페이지 테이블. 왜 페이지 테이블을 바로 안바꾸고 이지랄일까? 그 이유는 페이지 테이블 포맷이 담을 수 있는 정보의 한계가 있기 때문이다.
    - spt_find_page : spt와 가상 주소로부터 페이지 구조체를 찾는다
        
        Find VA from spt and return page
        
- 두가지 목적
    1. 페이지 폴트 시, 커널은 supplemental page table의 가상 페이지를 탐색하여 데이터가 거기에 있는지 찾는다.
    2. 프로세스가 종료됐을 때, 커널은 어떤 자원을 free할지 결정한다.

### Frame Table

할당/삭제된 **물리메모리 프레임 정보를 저장**하는 전역 데이터 스트럭쳐

- 모든 프레임이 가득 찼을 때, 쉽게 프레임을 얻기 위해서 테이블을 갖는다. 근데 프레임 가득 찼을 때 어떻게 프레임을 얻을까?
- 솔루션 : 프레임 테이블에 의해서 관리 하에 삭제가 이루어진다.
- 페이지 replacement algorithm은 LRU에 근접해야 하고 최소한 clock 알고리즘 만큼 수행해야 합니다.
    - vm_get_victim
- user_page를 위한 프레임만 관리한다(PAL_USER)
- 어떻게 페이지를 지울까?
    1. page replacement algorithm을 사용하여 지우기 위한 프레임을 선택한다.
    2. 프레임을 참조하는 페이지 테이블을 제거한다.(추가 크레딧을 구현하는 경우에만 여러 참조가 있다)
    3. 만약 필요하다면 페이지를 파일시스템이나 스왑 공간에 write한다.

### Swap Table

 스왑 슬롯의 정보를 담는다.

- 사용 추적과 스왑슬롯을 삭제
- 사용하지 않는 스왑 슬롯을 선택하여 페이지를 프레임에서 스왑 파티션으로 제거할 수 있게 함
- 페이지를 read back하거나 process 종료 시에 swap slot을 제거하게 한다. (프로세스별로 스왑 영역 비워져야)

 

### Synchronization 조심

여러 쓰레드들이 page in/page out 을 동시에 시도할 수 있다. 당신의 데이터 스트럭쳐가 동시성을 띄는지 확인해라

### Stack Growth

- project2 : 유저 프로세서의 스택에 pre-allocate 공간
- project3 : 필요에 따라 프로세스 스택에 더 많은 페이지들을 동적 할당
- valid한 스택 접근은 페이지 폴트를 발생시킬 수 있다!
- 만약 스택 접근이 유효하다면 페이지 폴트 핸들러에서 새로운 페이지를 할당한다.
- page_fault()에 전달된 구조체 intr_frame에서 %rsp 가져오기

### Memory Mapped Files

- mmap()과 munmap()
- 프로세스들은 그들의 가상주소공간에 file들을 맵핑할 것이다. 그때 memory-mapped 페이지들은 디스크로부터 lazily하게 로드 되어야한다.
- mmap()은 아래와 같을 때 error status을 리턴한다.
    - 파일의 사이즈가 0바이트일 때
    - 이미 맵핑된 페이지에 다른 파일이 오버랩 되려고 할때
    - 주소가 페이지 정렬되지 않았을 때
- 만약 mmap된 페이지를 삭제하려고 할 때, 변경 내용을 원래 파일에 다시 써야 한다.
- 모든 맵핑은 프로세스 종료시에 암묵적으로 unmapped 된다.

### 어디에 페이지 아웃/삭제?

- 다른 타입의 페이지들일 때 페이지 아웃될 수 있다.
- 유저스택 페이지들→ 스왑으로 페이지 아웃
- File pages(mmap된 파일들) → 파일시스템으로 페이지 아웃
    - 만약 dirty라면, 상응하는 파일에 변동사항을 write해야 한다.
    - 만약 not dirty라면, 단순히 deallocate하면 된다. 왜냐하면 파일시스템으로부터 다시 reload할 수도 있기 때문
