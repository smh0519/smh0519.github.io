---
title: "[Pintos - UserPrograme] - Argument Passing & System Call 핵심 구현"
date: 2025-12-03  # 작성 날짜에 맞춰 수정하세요
categories: [OS, Pintos, UserPrograme]
tags: [Operating System, Memory, Register]
---

Pintos Project 2의 첫 번째 목표인 **Argument Passing**을 성공적으로 마치고, args-\* 테스트들을 통과했다. 이 글에서는 단순히 개념만 짚는 것이 아니라, 실제 **어떤 함수를 어떻게 수정했고, 왜 그렇게 코드를 짰는지** 단계별로 상세히 리뷰해본다.

## 1단계: 문자열 파싱 (String Parsing)

사용자가 입력한 명령어가 /bin/ls -l이라면, 운영체제는 이를 있는 그대로 실행하는 것이 아니라 실행 파일 이름(/bin/ls)과 인자(-l)로 분리해야 한다. 이 작업은 커널 스레드가 생성된 직후, 실제 프로그램을 메모리에 올리기 직전인 process\_exec 함수에서 수행했다.

###  userprog/process.c - process\_exec 수정

기존 핀토스 코드는 파일 이름 하나만 load 함수에 넘겨줬지만, 이제는 strtok\_r을 이용해 문자열을 단어 단위로 자르고, 그 개수(count)와 배열(parse)을 load 함수에 넘겨주도록 변경했다.

**왜 이렇게 하나요?** load 함수는 디스크에서 실행 파일을 찾아 메모리에 올리는 역할을 한다. 만약 "ls -l"을 통째로 넘기면 OS는 "ls -l"이라는 이름의 (공백이 포함된) 파일을 찾으려 할 것이고, 결국 파일을 찾지 못해 실행에 실패하게 된다. 따라서 실행 파일명인 "ls"만 명확히 걸러내어 전달해야 한다.

```
int
process_exec (void *f_name) {
    char *file_name = f_name;
    bool success;

    /* 스레드 문맥 전환을 위한 인터럽트 프레임 선언 */
    struct intr_frame _if;
    _if.ds = _if.es = _if.ss = SEL_UDSEG;
    _if.cs = SEL_UCSEG;
    _if.eflags = FLAG_IF | FLAG_MBS;

    /* 1. 현재 프로세스 자원 정리 */
    process_cleanup ();

    /* -------------------------------------------------------- */
    /* 2. [추가] 명령어 파싱 (Argument Parsing) */
    /* "ls -l -a" -> ["ls", "-l", "-a"] 로 분리 */
    
    char *parse[64]; // 인자들을 저장할 배열 (최대 64개 가정)
    char *token, *save_ptr;
    int count = 0;

    // strtok_r을 이용해 공백 기준으로 문자열 분리
    for (token = strtok_r(file_name, " ", &save_ptr); token != NULL; token = strtok_r(NULL, " ", &save_ptr)) {
        parse[count++] = token;
    }
    /* -------------------------------------------------------- */

    /* 3. 바이너리 로드 (파싱된 인자들과 개수를 함께 전달!) */
    // 기존: success = load (file_name, &_if);
    // 수정: parse[0]이 실제 프로그램 이름이다.
    success = load (file_name, &_if, parse, count);

    /* 로드 실패 시 메모리 해제 및 종료 */
    palloc_free_page (file_name);
    if (!success)
        return -1;

    /* 4. 유저 프로그램 실행 시작 (Context Switch) */
    do_iret (&_if);
    NOT_REACHED ();
}
```

## 2단계: 유저 스택 쌓기 (Stack Construction)

이 프로젝트의 **하이라이트**이자 가장 까다로운 부분이다. x86-64 호출 규약(System V ABI)에 따라 \*\*유저 스택(User Stack)\*\*에 인자들을 정해진 순서대로 쌓아줘야 한다. 이 작업은 load 함수 내부에서 setup\_stack으로 빈 스택 공간이 만들어진 직후에 수행된다.

### 스택 메모리 구조 시각화 (/bin/ls -l foo bar 예시)

코드를 보기 전에, 최종적으로 만들어져야 할 스택의 모습을 먼저 이해해보자. 스택은 \*\*높은 주소에서 낮은 주소(High -> Low)\*\*로 자라기 때문에, 데이터가 위에서부터 아래로 쌓이는 구조다.

주소 (Address)변수명값 (Data)타입설명

| 0x4747fffc | argv\[3\]\[...\] | 'bar\\0' | char\[4\] | 문자열 데이터 |
| --- | --- | --- | --- | --- |
| 0x4747fff8 | argv\[2\]\[...\] | 'foo\\0' | char\[4\] | 문자열 데이터 |
| 0x4747fff5 | argv\[1\]\[...\] | '-l\\0' | char\[3\] | 문자열 데이터 |
| 0x4747ffed | argv\[0\]\[...\] | '/bin/ls\\0' | char\[8\] | 문자열 데이터 |
| 0x4747ffe8 | \- | 0 | uint8\_t\[\] | **Word Align** (8바이트 정렬을 위한 패딩) |
| 0x4747ffe0 | argv\[4\] | 0 | char \* | **NULL Pointer** (Sentinel) |
| 0x4747ffd8 | argv\[3\] | 0x4747fffc | char \* | bar 문자열의 주소를 가리킴 |
| 0x4747ffd0 | argv\[2\] | 0x4747fff8 | char \* | foo 문자열의 주소를 가리킴 |
| 0x4747ffc8 | argv\[1\] | 0x4747fff5 | char \* | \-l 문자열의 주소를 가리킴 |
| 0x4747ffc0 | argv\[0\] | 0x4747ffed | char \* | /bin/ls 문자열의 주소를 가리킴 |
| 0x4747ffb8 | return addr | 0 | void (\*)() | **Fake Return Address** |

위 표를 보면 알 수 있듯이, \*\*실제 문자열 데이터(Data)\*\*가 가장 깊숙한 곳(높은 주소)에 위치하고, 그 주소를 가리키는 \*\*포인터들(Pointers)\*\*이 그 아래에 위치한다. 그리고 CPU가 읽기 좋게 주소를 8의 배수로 맞추는 **정렬(Padding)** 과정이 중간에 들어간다.

###  userprog/process.c - argument\_stack 구현 (새로 추가)

위의 스택 구조를 코드로 구현하면 다음과 같다. 스택 포인터(rsp)를 직접 제어하여 데이터를 밀어 넣는다.

```
/* 인자들을 유저 스택에 쌓는 함수 */
void
argument_stack (char **parse, int count, struct intr_frame *if_) {
    char *argv_addrs[64]; // 인자 문자열들이 저장된 주소를 기억할 배열

    /* 1. 문자열 데이터 밀어 넣기 (Data Push) */
    // 스택의 맨 안쪽(높은 주소)부터 채워야 하므로 뒤에서부터 순회
    // 예: bar -> foo -> -l -> /bin/ls 순서
    for (int i = count - 1; i >= 0; i--) {
        int len = strlen(parse[i]) + 1; // 문자열 길이 + NULL 문자('\0')
        if_->rsp -= len;                // 스택 포인터 내리기 (공간 확보)
        memcpy((void *)if_->rsp, parse[i], len); // 확보된 공간에 문자열 복사
        argv_addrs[i] = (char *)if_->rsp; // ★중요: 방금 저장한 문자열의 주소를 따로 기록해둔다.
    }

    /* 2. Word Align (8바이트 정렬) */
    // 성능 향상을 위해 스택 주소를 8의 배수로 맞춤 (패딩 삽입)
    while ((uintptr_t)if_->rsp % 8 != 0) {
        if_->rsp--;
        *(uint8_t *)if_->rsp = 0; // 패딩(Padding) 0으로 채움
    }

    /* 3. 포인터 배열(argv) 밀어 넣기 */
    
    // 3-1. Sentinel (배열의 끝 표시)
    if_->rsp -= 8;
    *(char **)if_->rsp = NULL; // argv[argc] = NULL (C 표준 규약)

    // 3-2. 실제 주소들 (argv[argc-1] ... argv[0])
    // 위 표에서 argv[3] -> argv[2] -> ... 순서로 넣는 과정
    for (int i = count - 1; i >= 0; i--) {
        if_->rsp -= 8;
        *(char **)if_->rsp = argv_addrs[i]; // 아까 1번 단계에서 기록해둔 주소 사용
    }

    /* 4. 가짜 리턴 주소 (Fake Return Address) */
    // main 함수가 리턴할 곳은 없지만, 다른 함수 호출 규약과 맞추기 위해 0을 넣음
    if_->rsp -= 8;
    void (*ra)(void) = 0;
    *(void (**)(void))if_->rsp = ra;

    /* 5. 레지스터 세팅 (Register Setup) */
    // main(int argc, char *argv[]) 에게 인자 전달
    if_->R.rdi = count;                   // RDI = argc (인자 개수)
    if_->R.rsi = (uint64_t)if_->rsp + 8;  // RSI = argv (argv[0]의 주소)
    // 주의: 현재 rsp는 '가짜 리턴 주소'를 가리키므로 +8을 해야 argv 배열 시작점임
}
```

### userprog/process.c - load 함수 수정

load 함수는 이제 parse 배열과 count를 인자로 받아야 하며, 스택 생성 후 argument\_stack을 호출해야 한다.

```
/* 함수 원형 변경: char **argv, int argc 추가 */
static bool
load (const char *file_name, struct intr_frame *if_, char **argv, int argc) {
    // ... (ELF 헤더 파싱 및 세그먼트 로드 코드 생략) ...

    /* 스택 생성 (텅 빈 스택) */
    if (!setup_stack (if_))
        goto done;

    /* -------------------------------------------------------- */
    /* [추가] 생성된 스택에 인자 채워 넣기 */
    argument_stack(argv, argc, if_);
    /* -------------------------------------------------------- */

    /* 시작 주소 설정 (RIP) */
    if_->rip = ehdr.e_entry;

    success = true;
    // ... (이하 생략) ...
}
```

## 3단계: 시스템 콜 핸들러 (System Call Handler)

인자 전달이 잘 되었는지 확인하려면, 프로그램이 exit로 종료되거나 write로 화면에 출력을 할 수 있어야 한다. 유저 프로그램은 하드웨어에 직접 접근할 수 없으므로(Protection), 커널에게 "화면에 글씨 좀 써줘", "나 이제 죽여줘"라고 부탁하는 **시스템 콜**을 구현해야 한다.

###  userprog/syscall.c - 전체 구현 코드

가장 먼저 해야 할 일은 유저가 넘겨준 포인터(주소)가 안전한지 검사하는 것이다.

**왜 검사해야 하나요?** 악의적인 유저 프로그램이 커널 영역의 메모리를 참조하거나, NULL 포인터를 넘기면 OS 전체가 셧다운(Page Fault)될 수 있다. 운영체제는 어떤 상황에서도 죽지 않아야 하므로 철저한 검증이 필요하다.

```
/* 주소 유효성 검사 함수 */
void 
check_address(void *addr) {
    struct thread *curr = thread_current();
    /* 1. NULL 포인터 체크 */
    /* 2. 유저 영역(KERN_BASE 미만)인지 체크 */
    /* 3. 할당된 페이지인지(pml4_get_page) 체크 */
    if (addr == NULL || !is_user_vaddr(addr) || pml4_get_page(curr->pml4, addr) == NULL) {
        exit(-1); // 위반 시 프로세스 강제 종료
    }
}

/* 시스템 콜 핸들러 메인 */
void
syscall_handler (struct intr_frame *f UNUSED) {
    // RAX 레지스터에 시스템 콜 번호가 들어있다.
    int syscall_num = f->R.rax;

    switch (syscall_num) {
        case SYS_HALT:
            halt();
            break;
            
        case SYS_EXIT:
            // 인자 1개: status (RDI)
            exit(f->R.rdi);
            break;
            
        case SYS_WRITE:
            // 인자 3개: fd(RDI), buffer(RSI), size(RDX)
            // 반환값: RAX에 저장
            f->R.rax = write(f->R.rdi, (void *)f->R.rsi, f->R.rdx);
            break;
            
        default:
            exit(-1); // 정의되지 않은 시스템 콜
            break;
    }
}

/* SYS_EXIT 구현 */
void 
exit(int status) {
    struct thread *curr = thread_current();
    curr->exit_status = status; // 종료 상태 저장 (struct thread에 필드 추가 필요)
    
    // 필수 출력 로그 (채점 프로그램이 확인)
    printf("%s: exit(%d)\n", curr->name, status);
    thread_exit();
}

/* SYS_WRITE 구현 */
int 
write (int fd, const void *buffer, unsigned size) {
    check_address((void *)buffer); // 버퍼 주소가 유효한지 먼저 확인

    if (fd == 1) {
        // STDOUT (표준 출력)인 경우 화면에 출력
        putbuf(buffer, size);
        return size;
    } 
    // ... 파일 쓰기 등은 추후 구현 ...
    return 0;
}
```

---

## 4\. 정리 및 결과

이렇게 세 단계(파싱 -> 스택 적재 -> 시스템 콜 처리)를 거쳐 args-none, args-single 등의 7개의 테스트를 통과할 수 있었다.

-   **Argument Stack:** 포인터 연산과 스택 프레임 구조(Data -> Align -> Argv -> RetAddr)를 이해하는 것이 핵심이었다. 특히 **메모리 주소가 8바이트 단위로 정렬되어야 한다는 점**과 **리틀 엔디안** 방식의 메모리 적재 방식을 시각적으로 그려보며 디버깅했다.
-   **System Call:** 유저 프로그램은 스스로 아무것도 할 수 없으며, 반드시 커널의 도움(syscall)을 받아야 한다는 OS의 권한 분리 구조를 코드로 체감했다.
