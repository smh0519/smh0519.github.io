---
title: "[Pintos] Alarm 기능 구현기"
date: 2025-11-11
categories: [OS, Pintos]
---
스레드의 기본 개념을 공부하고 본격적으로 Pintos(핀토스) 구현에 들어갔다. 그런데… 이게 생각보다 너무 어렵다. 😅

특히 함수와 매크로를 이해하는 데만 **2~3일**이 걸렸다. Pintos는 단순히 코드만 짜는 게 아니라, 운영체제가 “어떻게 스레드를 관리하고, 언제 실행시킬지”를 직접 설계해야 하기 때문이다. 아직 구현은 손도 못 댄 상태에서, 하루 종일 list\_elem, thread\_block, intr\_disable 같은 이름만 들여다보고 있었다. “대체 얘네가 무슨 일을 하는 거지…?”그렇게 며칠을 헤맨 끝에, 드디어 첫 번째 목표였던 **Alarm(알람) 기능**을 구현하기 시작했다.

## Alarm

운영체제에서 **“알람(Alarm)”** 기능은 스레드(Thread)가 “일정 시간 동안 잠들었다가 자동으로 깨어나는 기능”을 말한다. 현재 구현되어 pintos에 구현되어 있는 방식은 busy-wait이다. 이 방식은 잠든 스레드에게 "너 지금 깨어있니?"라고 계속 물어보면서 만약 만들어있다면 다른 스레드에게 양보를 하는 방식이기 때문에 굉장히 비효율 적이라고 볼 수 있다. 

```
sleep -> wake up -> time check -> sleep -> wake up -> time check -> wake up
```

이러한 방법을 해결하기 위해서 한번 잠에 든 스레드는 일어나는 시간을 기억하고 그 전까지는 스케줄링에 포함 시키지 않는다면 조금 더 효율적으로 동작할 수 있을 것이다.

## 구현

#### 1) 리스트 선언 & 구조체 타입 추가

```
// 잠든 상태의 스레드들이 담긴 리스트
struct list sleep_list;
```

```
struct thread {
	...
	int64_t wait_time; /*깨어날 시간대를 확인*/
    ...
};
```

#### 2) 사용자가 잠을 요청

```
void timer_sleep (int64_t ticks) {
  int64_t start = timer_ticks();                 // 현재 절대시간(틱)
  ASSERT (intr_get_level() == INTR_ON);
  thread_sleep(start + ticks);                   // 절대시각으로 변환하여 위임
}
```

**nt64\_t start = timer\_ticks();**

-   현재까지 **운영체제 부팅 이후 흐른 전체 tick 수**를 읽습니다.
-   즉, “지금 시각이 몇 tick인지”를 알려주는 함수입니다.

 **ASSERT(intr\_get\_level() == INTR\_ON);**

-   **인터럽트가 켜져 있는 상태(INTR\_ON)** 인지 검사합니다.
-   왜냐하면 timer\_sleep()은 사용자 스레드가 호출하는 함수이기 때문에,  
    인터럽트를 꺼버리면 커널이 시간(tick)을 못 세게 되기 때문이에요.

> 💡 **핵심 포인트:**  
> Sleep 요청은 “시간이 흘러야” 의미가 있으므로,  
> 타이머 인터럽트가 꺼진 상태에서는 동작하면 안 됩니다.

**thread\_sleep(start + ticks);**

-   start는 “현재 시간”, ticks는 “얼마 동안 잘 건지(대기 시간)”
-   두 값을 더해서 **“언제 깨어나야 하는지(절대시각)”** 를 계산합니다.
-   그리고 그 값을 들고 **thread\_sleep()** 함수로 넘깁니다.  
    → 즉, “이 시각에 깨어나야 해!” 라고 OS에게 맡기는 과정이에요.

#### 2) 현재 스레드를 재우기

```
//sleep_list에 잠자는 스레드 추가
void thread_sleep(int64_t ticts){
	struct thread *cur;
	enum intr_level old_level;

	old_level = intr_disable();//인터랩트 중단
	cur =	thread_current(); // 현제 CPU에서 동작하는 스레드

	cur->wait_time = ticts;
	list_insert_ordered(&sleep_list, &cur->elem,c_thread_ticks,NULL);
	thread_block();// 스레드 생명주기에서 제외   

	intr_set_level(old_level);//인터랩트 활성화   
	
}
```

스레드를 재우고 블록 시키므로 자는 스레드를 스케줄링에서 제외시키는 함수이다. 이렇게 되면 전 코드보다 효율적으로 스레드를 관리 할 수 있게 된다.

**old\_level = intr\_disable();**

> **“임계 구역 보호용: 인터럽트 잠시 끈다.”**

-   인터럽트가 켜져 있으면, 지금 스레드를 리스트에 넣는 도중  
    타이머 인터럽트가 동시에 발생해서 리스트를 건드릴 수 있습니다.  
    → 데이터 불일치(race condition) 발생.
-   그래서 인터럽트를 잠시 꺼서 “다른 코드가 sleep\_list에 접근하지 못하게” 합니다.  
    (old\_level은 나중에 원상복구하기 위해 저장)

---

**cur = thread\_current();**

> **“지금 CPU에서 실행 중인 스레드가 누구인지 확인.”**

-   thread\_current()는 “현재 실행 중인 스레드”의 구조체 포인터를 반환합니다.
-   즉, 지금 **잠들려는 스레드 자기 자신**을 의미해요.

---

**cur->wait\_time = ticts;**

> **“언제 깨어나야 하는지 기록.”**

-   인자로 넘어온 ticts는 이미 timer\_sleep()에서 start + ticks로 계산된 **절대시각**입니다.
-   따라서 스레드 구조체 내부의 wait\_time에 “나 이 시각에 깨워줘!”라고 저장합니다.

---

**list\_insert\_ordered(&sleep\_list, &cur->elem, c\_thread\_ticks, NULL);**

> **“sleep\_list에 정렬된 순서로 삽입.”**

-   sleep\_list는 “잠든 스레드 목록”입니다.
-   list\_insert\_ordered()는 단순히 추가하는 게 아니라,  
    wait\_time이 작은 순서(즉, 더 빨리 깨어날 순서)로 리스트에 정렬 삽입해줍니다.

---

 **thread\_block();**

> **“현재 스레드를 BLOCKED 상태로 바꾼다.”**

-   스레드의 상태를 **RUNNING → BLOCKED** 로 변경합니다.
-   BLOCKED 상태는 CPU 스케줄링 대상에서 제외되므로,  
    이제 이 스레드는 실행되지 않고 **운영체제의 sleep\_list에 잠들어 있게 됩니다.**

---

**intr\_set\_level(old\_level);**

> **“인터럽트 상태 복원.”**

-   **old\_level = intr\_disable();** 에서 끈 인터럽트를 다시 원래 상태로 돌려놓습니다.
-   만약 원래 켜져 있었다면 다시 켜고, 꺼져 있었다면 그대로 둡니다.

#### 3) 재운 스레드 삽입 위치 정하기

```
bool c_thread_ticks(const struct list_elem *a, const struct list_elem *b, void *aux){
	struct thread* a1 = list_entry(a,struct thread,elem); // a
	struct thread* b1 = list_entry(b,struct thread,elem);
	return a1->wait_time < b1->wait_time;
}
```

**list\_insert\_ordered에** 인자 값으로 들어갈 함수로 **sleep\_list에** 들어갈 위치를 찾기 위해 사용된다.

동작 원리는 단순히 비교를 하는 것이지만 **list\_entry**라는 메크로를 이해하는데 조금 시간이 걸렸다**.**

#### 4) 자는 스레드 깨우기

```
void thread_wake_up(void){
	int64_t cur_time = timer_ticks ();

	while (!list_empty(&sleep_list))
	{
		struct thread* t = list_entry(list_front(&sleep_list),struct thread,elem);
		if(t->wait_time > cur_time){
			break;
		}
		list_pop_front(&sleep_list); // sleep_list에서 삭제
		thread_unblock(t); // 잠에서 깬 스래드 언브록
	}	
}
```

**int64\_t cur\_time = timer\_ticks ();**

-   **현재 커널 시간(틱)** 을 스냅샷으로 한 번 읽어 둔다.
-   이 값을 기준으로 “깰 시간(wait\_time)이 지났는가?”를 판단한다. (여러 번 timer\_ticks()를 부르면 도중에 값이 바뀔 수 있으니, 한 번 읽고 계속 쓰는 게 안전/일관적이다.)

---

**  
while (!list\_empty(&sleep\_list))**

-   **잠들어 있는 스레드들의 목록(sleep\_list)** 이 빌 때까지 확인한다.
-   sleep\_list는 “**언제 깰지(오름차순)**”로 정렬되어 있다는 전제가 있다. (가장 먼저 깨어날 스레드가 항상 맨 앞)

---

**struct thread\* t = list\_entry(list\_front(&sleep\_list), struct thread, elem);**

-   리스트 **맨 앞(front)** 의 노드를 집어 와서, 그 노드가 속한 **스레드 구조체(struct thread)** 로 복원한다.
-   맨 앞은 “**가장 먼저 깨어날**” 스레드이므로, 이 스레드부터 깰지 말지 판단한다.

  
이렇게 구현이 끝나고 테스트 케이스를 돌려보면

[테스트 이미지](/assets/img/2025-11-11/image1.png)
성공!!!

