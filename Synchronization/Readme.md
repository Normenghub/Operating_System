# 🔒 운영체제 - 동기화 (Synchronization)
 

 
---
 
## 📌 목차
 
1. [쓰레드 (Thread)](#-쓰레드-thread)
2. [동시성 문제](#-동시성-문제)
3. [Lock](#-lock)
4. [Lock 구현 방법들](#-lock-구현-방법들)
5. [스핀 없이 대기하기](#-스핀-없이-대기하기)
6. [Condition Variable](#-condition-variable-조건-변수)
7. [Semaphore](#-semaphore-세마포어)
8. [Deadlock](#-deadlock-교착-상태)
9. [고전 동기화 문제들](#-고전-동기화-문제들)
---
 
## 🧵 쓰레드 (Thread)
 
프로세스 간 컨텍스트 스위칭의 오버헤드가 너무 비싸다 → 그래서 한 프로세스 안에 **멀티쓰레드**를 넣어 활용하기 시작했다.
 
### 쓰레드가 각자 가지는 것
- 쓰레드 ID
- 프로그램 카운터 (PC)
- 스택 포인터
- 스택
- 레지스터
### 쓰레드끼리 공유하는 것 (같은 프로세스 내)
- 코드 (Code)
- 데이터 (Data)
- 파일 (Files)
- 힙 (Heap) 등 동적 객체
> 📌 지역 변수는 공유 안 함 → 각 쓰레드가 자신의 스택을 가지기 때문  
> 글로벌 변수, 동적 객체(힙)는 공유됨
 
### 쓰레드 스위칭 단계
 
```
T1 쓰레드 레지스터 상태 저장 → T2 레지스터 상태 복원
어드레스 스페이스는 유지 (같은 프로세스 내 쓰레드이기 때문)
```
 
### 멀티쓰레딩의 이익
- 프로그램 구조 향상
- 처리량(Throughput) 향상
- 응답성 향상
- 자원 공유
- 병렬 프로그래밍 가능
- 동시성 생성 비용이 쌈
### 쓰레드 개수 분류
 
| 단일 쓰레드 | 다수 쓰레드 |
|------------|------------|
| MS/DOS, 초기 Macintosh, Unix xv6 | 많은 임베디드 OS, macOS, Linux, Windows, Solaris |
 
### 쓰레드 라이브러리
 
API로 프로그래머에게 쓰레드 컨트롤을 제공해줌. 크게 두 종류로 나뉜다.
 
#### 커널 레벨 쓰레드 (Kernel-level Thread)
 
- OS가 쓰레드와 프로세스를 관리
- 쓰레드 관련 오퍼레이션이 **커널 모드**에서 구현됨
- 쓰레드 생성/관리 시 **시스템 콜** 필요
- OS가 쓰레드들을 스케줄링
- 쓰레드 생성 비용 < 프로세스 생성 비용
- 예: Windows, Linux, macOS, Solaris
**제한점**
- 모든 오퍼레이션이 시스템 콜을 통해 진행 → 생성·관리 비용이 비쌈
- 동시 쓰레드 수에 제한이 있을 수 있음
- 쓰레드 수 증가에 따라 OS의 확장성이 뛰어나야 함
- 모든 프로그래머·언어·런타임 시스템에 범용적이어야 함
#### 유저 레벨 쓰레드 (User-level Thread)
 
- 이식성이 높고 작고 빠름
- 커널 환경에서 오퍼레이션이 진행되지 않음
- 쓰레드는 OS가 볼 수 없음
- 프로그램에 연결된 **라이브러리**가 쓰레드를 관리
- 애플리케이션 요구사항에 맞게 조정 가능
**제한점**
- 보통 **비선점형 스케줄링**에 의존
- OS가 잘못된 결정을 할 수 있음 (예: I/O 블로킹으로 프로세스 전체가 먹통)
- 쓰레드가 락을 쥔 상태에서 해당 쓰레드를 포함한 프로세스가 교체되면 먹통
- **멀티코어 CPU 활용 불가**
---
 
## ⚡ 동시성 문제
 
쓰레드 간 데이터 공유 시 각 실행마다 다른 결과가 나올 수 있다 → **비결정론적(Non-deterministic)**
 
| 개념 | 설명 |
|------|------|
| **Race Condition** | 결과가 실행 타이밍에 의존하는 것. 비결정론적 결과 초래 |
| **Critical Section** | 공유 변수에 접근하는 코드 영역. 원자적으로 실행되어야 함 |
| **Mutual Exclusion** | Critical Section을 원자적으로 만들어주는 것 |
 
공유 자원 접근 제어를 위해 **동기화 메커니즘**이 필요하다.
 
---
 
## 🔐 Lock
 
### 기본 아이디어
 
MUTEX 변수를 자물쇠라고 하자. `balance += 1` 같은 코드 실행 시 아무도 간섭 못하게 위아래로 락을 건다.
 
```c
lock(&mutex);
balance = balance + 1;
unlock(&mutex);
```
 
### Lock 변수의 두 가지 상태
 
| 상태 | 의미 |
|------|------|
| Available (free / unlocked) | 들어와도 됨 |
| Acquired (locked / held) | 지금 누가 쓰고 있음 |
 
### lock() / unlock() 함수
 
- `lock()` : Critical Section으로 들어감. 상태를 Acquired로 만듦 (Free 상태여야 진입 가능)
- `unlock()` : Lock 주인이 호출하면 Available 상태로 변경. 대기 중인 쓰레드를 깨워줌
### Lock의 평가 기준
 
- **공정성(Fairness)** : 각 쓰레드가 락을 획득할 공정한 기회를 얻는가
- **정확성(Correctness)** : 상호 배제 (임계 영역에 한 번에 하나의 쓰레드만), 진행성 (하나만 진입 가능), Bounded Waiting (기아 상태 방지)
- **성능(Performance)** : 경쟁(contention)이 없을 때와 있을 때(멀티 CPU 포함)의 오버헤드
> 💡 락을 아무도 경쟁하지 않을 때(no contention)는 오버헤드가 작고, 여러 쓰레드가 동시에 락을 획득하려 경쟁할 때(with contention)는 오버헤드가 커진다. 멀티 CPU 환경에서는 이 비용이 달라진다.
 
---
 
## 🔧 Lock 구현 방법들
 
### 1. Controlling Interrupts (인터럽트 제어)
 
싱글 프로세서를 위해 발명됨. 락 상태와 무관하게 단순히 인터럽트를 끄고 킴.
 
```c
void lock() {
    DisableInterrupts();
}
void unlock() {
    EnableInterrupts();
}
```
 
**문제점**
- 악의적인 프로그램이 프로세서를 독점 가능
- 멀티 프로세서에서는 동작하지 않음
---
 
### 2. Peterson's Algorithm (소프트웨어 알고리즘)
 
- `turn` : 누구 차례인지
- `flag[2]` : 누가 사용 중인지
```c
int flag[2];
int turn;
 
void init() {
    flag[0] = flag[1] = 0;  // 1 → thread wants to grab lock
    turn = 0;               // whose turn? (thread 0 or 1?)
}
 
void lock() {
    int other = 1 - self;
    flag[self] = 1;         // 나 쓰고 싶음
    turn = other;           // 상대방 차례로 양보
    while ((flag[other] == 1) && (turn == other));  // spin-wait
}
 
void unlock() {
    flag[self] = 0;
}
```
 
---
 
### 3. 왜 하드웨어 지원이 필요한가?
 
```c
void lock(lock_t *mutex) {
    while (mutex->flag == 1)  // TEST
        ;                     // spin-wait
    mutex->flag = 1;          // SET
}
```
 
**문제** : spin-wait 중 인터럽트로 다른 쓰레드로 스위칭되면? → 둘 다 `flag = 1`로 세팅될 수 있음. 또한 스핀 웨이팅은 시간 낭비.
 
→ **하드웨어적으로 원자성을 보장**해야 함.
 
---
 
### 4. Test-And-Set (Atomic Exchange)
 
```c
int TestAndSet(int *ptr, int new) {
    int old = *ptr;   // 이전 값 읽기
    *ptr = new;       // 새 값 쓰기
    return old;       // 이전 값 반환
}
// 위 함수는 원자적으로(끊김 없이) 실행된다.
```
 
```c
typedef struct __lock_t { int flag; } lock_t;
 
void init(lock_t *lock) {
    lock->flag = 0;  // 0 = available, 1 = held
}
 
void lock(lock_t *lock) {
    while (TestAndSet(&lock->flag, 1) == 1)
        ;  // spin-wait
}
 
void unlock(lock_t *lock) {
    lock->flag = 0;
}
```
 
`flag`가 0일 때 spin-wait 종료되고 flag를 1로 세팅.
 
#### 스핀 락 평가
 
| 기준 | 결과 |
|------|------|
| 정확성 | ✅ YES - 한 쓰레드만 진입 가능 |
| 공평성 | ❌ NO - 어떤 쓰레드는 영원히 spin-wait 가능 (기아 발생) |
| 성능 | 단일 CPU에서 오버헤드 큼. 쓰레드 수 ≈ CPU 수이면 잘 작동 |
 
---
 
### 5. Compare-And-Swap (CAS)
 
```c
int CompareAndSwap(int *ptr, int expected, int new) {
    int actual = *ptr;
    if (actual == expected)
        *ptr = new;
    return actual;
}
 
void lock(lock_t *lock) {
    while (CompareAndSwap(&lock->flag, 0, 1) == 1)
        ;  // spin
}
```
 
`expected` 값과 현재 값을 비교해서 값이 바뀌었는지 체크 가능.
 
---
 
### 6. Load-Linked / Store-Conditional (LL/SC)
 
**CAS의 한계 - ABA 문제** : 값이 0→1→0으로 바뀌었다가 다시 돌아왔을 때 CAS는 이를 감지하지 못함. LL/SC는 이를 해결한다.
 
```c
int LoadLinked(int *ptr) {
    return *ptr;
}
 
int StoreConditional(int *ptr, int value) {
    // LoadLinked 이후 *ptr이 변경되지 않았다면
    if (no update since LoadLinked) {
        *ptr = value;
        return 1;  // success
    } else {
        return 0;  // failed
    }
}
 
void lock(lock_t *lock) {
    while (1) {
        while (LoadLinked(&lock->flag) == 1)
            ;  // flag가 0이 될 때까지 spin
        if (StoreConditional(&lock->flag, 1) == 1)
            return;  // 성공 시 종료
        // 실패 시 처음부터 다시
    }
}
 
void unlock(lock_t *lock) {
    lock->flag = 0;
}
```
 
무한 루프 안에서 `LoadLinked`로 flag가 0이 되면 while 탈출 → `StoreConditional`로 그 사이 값이 바뀌었는지 확인 → 성공하면 flag를 1로 업데이트, 실패하면 다시 처음부터.
 
---
 
### 7. Fetch-And-Add / Ticket Lock
 
```c
int FetchAndAdd(int *ptr) {
    int old = *ptr;
    *ptr = old + 1;
    return old;
}
```
 
increment 명령을 원자적으로 구현. 사실상 순서(번호표)를 알려줌.
 
#### Ticket Lock — 완전한 공정성 보장
 
대기표(ticket)를 주고 `turn`이 돌아오면 실행.
 
```c
typedef struct __lock_t {
    int ticket;
    int turn;
} lock_t;
 
void lock_init(lock_t *lock) {
    lock->ticket = 0;
    lock->turn = 0;
}
 
void lock(lock_t *lock) {
    int myturn = FetchAndAdd(&lock->ticket);  // 대기표 받기
    while (lock->turn != myturn)
        ;  // 내 번호 올 때까지 spin
}
 
void unlock(lock_t *lock) {
    FetchAndAdd(&lock->turn);  // 다음 번호로 넘김
}
```
 
`FetchAndAdd`는 `turn`과 `ticket` 둘 다에 사용되는 모듈. 원자성을 하드웨어적으로 구현해놓으면 사용 가능.
 
---
 
## 😴 스핀 없이 대기하기
 
스핀 락은 대기 중에도 CPU를 낭비한다 → 어떻게 해결할까?
 
### 방법 1: Just Yield
 
```c
void lock() {
    while (TestAndSet(&flag, 1) == 1)
        yield();  // CPU 양보
}
```
 
**단점** : 컨텍스트 스위칭 비용이 비쌈. 기아 발생 문제는 여전히 존재.
 
---
 
### 방법 2: 큐 + Sleep (park/unpark)
 
- `park()` : 쓰레드 잠들게 하기
- `unpark(threadID)` : 특정 쓰레드 깨우기
```c
typedef struct __lock_t {
    int flag;
    int guard;
    queue_t *q;
} lock_t;
 
void lock(lock_t *m) {
    while (TestAndSet(&m->guard, 1) == 1)
        ;  // guard 락 획득 (spin)
    if (m->flag == 0) {
        m->flag = 1;   // 락 획득
        m->guard = 0;
    } else {
        queue_add(m->q, gettid());  // 대기 큐에 등록
        m->guard = 0;
        park();                     // 잠들기
    }
}
 
void unlock(lock_t *m) {
    while (TestAndSet(&m->guard, 1) == 1)
        ;
    if (queue_empty(m->q))
        m->flag = 0;                      // 아무도 없으면 락 해제
    else
        unpark(queue_remove(m->q));       // 다음 쓰레드 깨움 (flag=1 유지!)
    m->guard = 0;
}
```
 
- `unlock()`에서 flag를 0으로 바꾸지 않고 1 유지 → 0으로 바꾸는 도중 다른 쓰레드로 스위칭되면 위험하기 때문
#### Wakeup/Waiting Race 문제
 
큐 등록 → `park()` 호출 **사이에** 쓰레드 스위칭이 생기고 `unpark()` 신호가 먼저 오면?
→ 아직 잠들지도 않았는데 시그널이 허공으로 사라짐. 다시 돌아와 `park()`를 호출하면 영원히 잠듦.
 
**해결책 (Solaris)** : `setpark()` 사용. 큐 등록 직후 호출하면 이후에 `unpark()` 신호가 먼저 와도 기록해두고, 이후 `park()` 호출 시 잠들지 않고 즉시 리턴.
 
---
 
### 방법 3: Two-Phase Lock (현대 OS)
 
스핀 락 + 슬리핑 큐 락의 혼합 버전.
 
1. **Phase 1** : 락을 누가 쥐고 있으면 **짧은 시간 동안 스핀**하며 풀리길 기다림
2. **Phase 2** : 스핀 후에도 락을 획득 못하면 **대기 큐에 잠들게** 함
---
 
## 📣 Condition Variable (조건 변수)
 
### 개념
 
- **Waiting condition** : 실행 상태가 원하는 대로 되지 않을 때 쓰레드 스스로 대기할 수 있는 명시적 대기열
- **Signaling condition** : 다른 쓰레드가 상태를 변경하면 대기 중인 쓰레드 중 하나를 깨워 실행할 수 있도록 함
### API
 
```c
pthread_cond_wait(pthread_cond_t *c, pthread_mutex_t *m);  // wait()
pthread_cond_signal(pthread_cond_t *c);                    // signal()
```
 
- `wait()` : 뮤텍스를 확보하고, 진행이 안 되는 조건을 만나면 **락을 해제**하고 대기 상태로 전환. 깨어나면 락을 다시 획득.
- `signal()` : 조건 변수를 기다리는 쓰레드가 있으면 그 쓰레드를 깨움.
### 예제: Parent waiting for Child
 
```c
int done = 0;
pthread_mutex_t m = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t c = PTHREAD_COND_INITIALIZER;
 
void thr_exit() {
    Pthread_mutex_lock(&m);
    done = 1;
    Pthread_cond_signal(&c);    // 부모 깨우기
    Pthread_mutex_unlock(&m);
}
 
void *child(void *arg) {
    printf("child\n");
    thr_exit();
    return NULL;
}
 
void thr_join() {
    Pthread_mutex_lock(&m);
    while (done == 0)
        Pthread_cond_wait(&c, &m);  // 잠듦 (락 해제 후 대기)
    Pthread_mutex_unlock(&m);
}
 
int main(int argc, char *argv[]) {
    printf("parent: begin\n");
    pthread_t p;
    Pthread_create(&p, NULL, child, NULL);
    thr_join();
    printf("parent: end\n");
    return 0;
}
```
 
**실행 흐름**
1. `main` 시작 → child 쓰레드 생성
2. `thr_join()` → 락 걸고 `done == 0`이면 `wait()` 호출 → 잠듦 (락 해제됨)
3. child 실행 → "child" 출력 → `thr_exit()` → `done = 1`, signal로 부모 깨움
4. 부모 깨어남 → 락 재획득 → unlock → "parent: end" 출력
> 컨텍스트 스위칭 순서가 달라도 OK: `thr_join()` 전에 child가 먼저 실행되면 `done = 1`이 되고, 이후 부모가 `wait()`를 호출해도 `done != 0`이므로 while문을 타지 않고 바로 리턴.
 
---
 
## 🚦 Semaphore (세마포어)
 
세마포어는 lock과 condition variable 역할을 모두 할 수 있다.
 
### API
 
```c
sem_t s;
sem_init(&s, 0, 1);
// 두 번째 인자 0 : 같은 프로세스 내 쓰레드들끼리 세마포어 공유
// 세 번째 인자 : 세마포어 초기값
```
 
#### sem_wait() — P 연산
세마포어 값을 **decrement**. 결과값이 **음수면 대기 상태** 전환.
- 값이 -3이면: 대기 중인 쓰레드가 3개 있다는 뜻
#### sem_post() — V 연산
세마포어 값을 **increment**. 값을 올리기 전에 대기 중인 쓰레드가 있으면(값이 음수면) 하나를 깨워 자원을 쓸 수 있게 함.
 
### 세마포어의 두 종류
 
| 종류 | 값 범위 | 용도 |
|------|---------|------|
| 이진 세마포어 (Binary) | 0 또는 1 | Mutex처럼 사용 |
| 카운팅 세마포어 (Counting) | 0 ~ N | 자원 개수 관리 |
 
### 세마포어 초기값
 
- **락(Mutex)으로 사용** : 초기값 **1**
- **순서 동기화(부모가 자식 기다릴 때)** : 초기값 **0**
### 예제: 부모가 자식을 기다리는 경우
 
```c
sem_t s;
 
void *child(void *arg) {
    printf("child\n");
    sem_post(&s);   // 완료 신호
    return NULL;
}
 
int main(int argc, char *argv[]) {
    sem_init(&s, 0, 0);   // 초기값 0 → 부모가 즉시 잠듦
    printf("parent: begin\n");
    pthread_t c;
    pthread_create(&c, NULL, child, NULL);
    sem_wait(&s);         // 자식 완료까지 대기
    printf("parent: end\n");
    return 0;
}
```
 
- 부모가 먼저 실행 : `sem_wait()` → 값 -1 → 잠듦 → child 실행 → `sem_post()` → 부모 깨움
- child가 먼저 실행 : `sem_post()` → 값 1 → 부모가 `sem_wait()` → 즉시 통과
---
 
## ⚠️ Deadlock (교착 상태)
 
서로 상대방이 가진 자원을 기다리며 **무한히 대기 상태에 빠지는 현상**.
 
### 예시
 
세마포어 S, Q가 모두 1인 상황:
 
| 단계 | T1 | T2 |
|------|----|----|
| 1 | S 획득 (S = 0) | |
| 2 | | Q 획득 (Q = 0) |
| 3 | Q 획득 시도 → Q = -1 → **잠듦** | |
| 4 | | S 획득 시도 → S = -1 → **잠듦** |
| 결과 | 💀 Deadlock | 💀 Deadlock |
 
### 기아 상태 (Starvation)
 
공정성이 보장되지 않으면 특정 쓰레드가 영원히 대기 큐에 갇혀버림.
 
### 우선순위 역전 (Priority Inversion)
 
우선순위 A > B > C인 상황:
 
1. C가 먼저 실행되어 자원(세마포어) 획득
2. B로 스위칭 → B 실행
3. A가 실행되려 하는데, A가 필요한 자원을 C가 쥐고 있음 → A 대기
4. 결과: A보다 낮은 우선순위의 B가 A보다 먼저 실행됨 → **우선순위 역전**
**해결책 (Priority Inheritance)** : 자원을 쥔 C의 우선순위를 일시적으로 A 수준으로 높여 C가 빨리 실행·완료되도록 함. C가 자원을 해제하면 우선순위 원상복귀, A 실행.
 
---
 
## 📚 고전 동기화 문제들
 
### 1. Bounded-Buffer Problem (유한 버퍼 문제)
 
**실제 예** : 웹 서버에서 클라이언트 요청을 큐에 쌓아놓고 작업 쓰레드들이 꺼내 처리하는 구조.
 
- **Producer** : `put()` — 데이터를 만들어 버퍼에 넣음. 버퍼가 꽉 차면 기다림.
- **Consumer** : `get()` — 버퍼에서 데이터 꺼냄. 버퍼가 비면 기다림.
```c
int buffer[MAX];
int fill = 0;  // 생산자가 넣을 위치
int use  = 0;  // 소비자가 꺼낼 위치
 
void put(int value) {
    buffer[fill] = value;
    fill = (fill + 1) % MAX;
}
 
int get() {
    int tmp = buffer[use];
    use = (use + 1) % MAX;
    return tmp;
}
```
 
생산자와 소비자가 동시에 `fill`, `use`를 사용하면 꼬임 → **세마포어로 해결**
 
세마포어 변수:
- `empty` : 버퍼의 빈 칸 개수 (초기값 = MAX)
- `full` : 버퍼에 채워진 데이터 개수 (초기값 = 0)
- `mutex` : 상호 배제용 (초기값 = 1)
```c
// Producer
for (i = 0; i < loops; i++) {
    sem_wait(&empty);  // 빈 자리 기다림
    sem_wait(&mutex);  // 락 걸기
    put(i);            // 임계 구역
    sem_post(&mutex);  // 락 해제
    sem_post(&full);   // 채워진 칸 +1 알림
}
 
// Consumer
for (i = 0; i < loops; i++) {
    sem_wait(&full);   // 데이터 생길 때까지 기다림
    sem_wait(&mutex);  // 락 걸기
    int tmp = get();   // 임계 구역
    sem_post(&mutex);  // 락 해제
    sem_post(&empty);  // 빈 자리 +1 알림
    printf("%d\n", tmp);
}
```
 
---
 
### 2. Reader-Writer Problem (독자-저자 문제)
 
읽기는 여러 명이 동시에 해도 되지만, 쓰기는 반드시 한 명만 해야 하는 상황.
 
- **Writer (Insert)** : 오직 한 명만 진입 가능 (Critical Section). 리스트 상태를 바꿈.
- **Reader (Lookup)** : 진행 중인 Write가 없다면 수백 수천 명이 동시에 읽어도 됨.
- 모든 Reader가 끝나기 전까지 Writer는 쓸 수 없음.
```c
typedef struct rwlock_t {
    sem_t lock;       // readers 카운터 보호용 (초기값 1)
    sem_t writelock;  // 실제 데이터 쓰기 락 (초기값 1)
    int readers;      // 현재 읽고 있는 Reader 수
} rwlock_t;
 
void rwlock_acquire_readlock(rwlock_t *rw) {
    sem_wait(&rw->lock);
    rw->readers++;
    if (rw->readers == 1)
        sem_wait(&rw->writelock);  // 첫 번째 Reader → Writer 락 잠금
    sem_post(&rw->lock);
}
 
void rwlock_release_readlock(rwlock_t *rw) {
    sem_wait(&rw->lock);
    rw->readers--;
    if (rw->readers == 0)
        sem_post(&rw->writelock);  // 마지막 Reader → Writer 락 해제
    sem_post(&rw->lock);
}
 
void rwlock_acquire_writelock(rwlock_t *rw) {
    sem_wait(&rw->writelock);
}
 
void rwlock_release_writelock(rwlock_t *rw) {
    sem_post(&rw->writelock);
}
```
 
**문제점** : Reader가 계속 들어오면 Writer가 영원히 대기 (기아 상태).
 
**해결책** : Writer가 대기 줄에 서는 순간, 그 뒤로 오는 Reader들은 방에 못 들어가고 대기 큐에 줄을 서게 만듦.
 
---
 
### 3. Dining Philosophers (식사하는 철학자 문제)
 
원형 테이블에 5명의 철학자, 포크 5개 (각 철학자 사이 1개씩).
 
- `think()` : 포크 필요 없음
- `eat()` : 왼쪽과 오른쪽 포크 **2개를 동시에** 쥐어야 함
**해결 조건** : 데드락 없음 / 기아 상태 없음 / 높은 동시성
 
```c
int left(int p)  { return p; }
int right(int p) { return (p + 1) % 5; }
```
 
#### 순진한 해법 (데드락 발생)
 
```c
void getforks() {
    sem_wait(forks[left(p)]);   // 왼쪽 포크 집기
    sem_wait(forks[right(p)]);  // 오른쪽 포크 집기
}
```
 
**문제** : 5명 모두 왼쪽 포크를 집으면 → 전원 오른쪽 포크를 기다리며 **데드락**.
 
#### 해결책: Breaking The Dependency (대칭성 파괴)
 
```c
void getforks() {
    if (p == 4) {
        // 마지막 철학자(4번)만 반대 순서로 집음
        sem_wait(forks[right(p)]);  // 오른쪽 먼저
        sem_wait(forks[left(p)]);   // 왼쪽 나중
    } else {
        sem_wait(forks[left(p)]);   // 왼쪽 먼저
        sem_wait(forks[right(p)]);  // 오른쪽 나중
    }
}
 
void putforks() {
    sem_post(forks[left(p)]);
    sem_post(forks[right(p)]);
}
```
 
한 명이 반대 순서로 포크를 집으면 순환 대기가 깨짐 → 데드락 해결.
 
---
 
## 참고: 리눅스 커널 선점형 변화
 
| 버전 | 방식 |
|------|------|
| Linux Kernel 2.6 이전 | 인터럽트 비활성화 방식 |
| Linux Kernel 2.6 이후 | 완전 선점형 (Fully Preemptive) |
 
---
