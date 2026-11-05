# Day 08 학습 기록 - 스마트 포인터와 소유권

- 진행 기간: 2026-10-16 ~ 2026-11-05
- 진행 방식: 대화형 질답
- 결과: **기본 소유권 선택 통과(🟢), 제어 블록과 관찰자 수명 보완(🟡)**
- 다음 복습: 2026-11-06, 2026-11-12, 2026-12-05

## 오늘 다룬 내용

- `unique_ptr`, `shared_ptr`, `weak_ptr`의 소유권 의미
- 강한 참조 수와 제어 블록
- 순환 참조와 `weak_ptr::lock()`
- 단독 소유 객체의 비소유 관찰자
- 함수 매개변수로 소유권 의도 표현하기
- `get()`, `release()`, `reset()`
- 같은 원시 포인터로 제어 블록을 중복 생성하는 문제
- `make_shared`의 장점
- `enable_shared_from_this`와 `shared_from_this()`
- `shared_ptr`의 스레드 안전성 범위
- `shared_ptr` 남용을 피해야 하는 이유

## 질답 기록

### Q1. 세 스마트 포인터는 어떤 소유 관계를 표현하는가?

**학습자 답변**

- `unique_ptr`: 외부에 소유권을 넘기지 않고 객체 소멸 시 메모리 해제
- `shared_ptr`: 소유권을 공유하고 모든 공유 객체가 사라지면 해제
- `weak_ptr`: 소유권을 빌려오지만 메모리 소멸 권한은 없음

**평가: 🟡**

`unique_ptr`는 소유권을 공유하지 않지만 `std::move`로 이전할 수 있다. `shared_ptr`는 같은 객체를 공동 소유하는 강한 참조 수를 관리한다. `weak_ptr`는 소유권을 받지 않으며 강한 참조 수를 늘리지 않는다.

```text
unique_ptr → 한 번에 하나만 소유, 이전 가능
shared_ptr → 여러 주체가 공동 소유
weak_ptr   → 소유하지 않고 관찰
```

### Q2. 강한 참조 수와 `weak_ptr::lock()`

```cpp
auto a = std::make_shared<Player>();
std::weak_ptr<Player> w = a;

{
    auto b = a;
}

a.reset();
auto c = w.lock();
```

**학습자 답변**

강한 참조 수를 `1 → 1 → 2 → 1 → 0 → 0`으로 정확히 추적하고 `a.reset()`에서 객체가 파괴된다고 답했다.

**평가: 🟢**

마지막 강한 소유자가 사라지면 객체가 파괴된다. 이후 `lock()`은 빈 `shared_ptr`를 반환한다. `weak_ptr`가 남아 있으면 제어 블록은 잠시 유지될 수 있지만 관리 대상 객체는 이미 파괴됐다.

### Q3. `shared_ptr` 순환 참조

```cpp
class Player { std::shared_ptr<Guild> guild; };
class Guild  { std::shared_ptr<Player> leader; };
```

**학습자 답변**

양쪽이 서로를 소유해 지역 변수가 사라져도 참조 수가 `2 → 1`에 머물며 메모리가 해제되지 않는다고 설명했다. 해결 수단으로 `weak_ptr`를 제시했다.

**평가: 🟢**

양쪽을 모두 `weak_ptr`로 바꾸는 것이 아니라, 소유권 방향을 결정한 뒤 **소유하지 않는 역참조 하나**를 `weak_ptr`로 바꾼다.

### Q4. `weak_ptr`로 `unique_ptr`를 관찰할 수 있는가?

**학습자 답변**

`unique_ptr`는 복사할 수 없으며, `Game`이 `shared_ptr`로 소유하면 `HUD`가 `weak_ptr`로 참조할 수 있다고 답했다.

**평가: 🟡**

`weak_ptr`는 `shared_ptr`의 제어 블록을 관찰하므로 `unique_ptr`를 직접 관찰할 수 없다. `Game`이 `Player`를 단독 소유하고 수명 순서를 보장한다면 `HUD`는 `Player*` 같은 비소유 원시 포인터를 사용할 수 있다. 이 포인터는 삭제해서는 안 되며 소유자가 먼저 파괴되면 댕글링 포인터가 된다는 조건이 따른다.

### Q5. 함수 매개변수로 소유권을 어떻게 표현하는가?

```cpp
void DrawPlayer(const Player& player);
void SetMainPlayer(std::unique_ptr<Player> player);
void RegisterPlayer(std::shared_ptr<Player> player);
```

**학습자 답변**

- 읽기만 함: `const Player&`
- 단독 소유권을 넘겨받음: `unique_ptr<Player>`
- 수명을 공동 관리함: `shared_ptr<Player>`

**평가: 🟢**

`unique_ptr`는 호출자가 `std::move`로 넘긴다. `shared_ptr`를 값으로 받으면 함수가 강한 소유권을 보관한다는 의도를 표현한다. 객체가 없을 수 있는 비소유 접근이라면 포인터를 고려할 수 있다.

### Q6. `get()`, `release()`, `reset()`의 차이는?

**학습자 답변**

`get()`은 삭제하면 안 되는 원시 포인터를 반환하고, `reset()`은 명시적으로 메모리를 해제한다고 답했다. `release()`는 처음 접했다.

**평가: 🟡**

```text
get()     → 주소만 빌려준다. 소유권 유지
release() → 주소를 반환하고 소유권을 포기한다. 삭제하지 않음
reset()   → 현재 소유 중인 객체를 삭제하고 소유권을 비운다
```

`release()` 이후에는 반환된 원시 포인터를 다른 소유자에게 넘기거나 직접 삭제하지 않으면 누수가 발생한다.

### Q7. 같은 원시 포인터로 `shared_ptr` 두 개를 만들면?

```cpp
Player* raw = new Player;
std::shared_ptr<Player> a(raw);
std::shared_ptr<Player> b(raw);
```

**학습자 답변**

각 참조 수가 1이라는 점은 맞혔지만, 원시 포인터를 별도로 삭제하지 않아 누수가 발생한다고 판단했다.

**평가: 🟡**

`a`와 `b`는 같은 주소에 대해 서로 다른 제어 블록을 만든다. 둘 다 자신이 유일한 소유자라고 판단해 함수 종료 시 같은 객체를 두 번 삭제한다. 올바른 방식은 하나의 `shared_ptr`를 만든 뒤 복사하는 것이다.

```cpp
auto a = std::make_shared<Player>();
auto b = a;
```

### Q8. `make_shared`를 권장하는 이유는?

**학습자 답변**

RAII 원칙에 따라 `make_shared`가 더 안전할 것으로 추론했다.

**평가: 🟡**

두 방식 모두 생성이 완료되면 RAII로 관리된다. `make_shared`는 보통 객체와 제어 블록을 한 번에 할당하고, 원시 포인터 노출을 줄이며, 코드가 간결하다.

> RAII: Resource Acquisition Is Initialization. 객체 수명에 자원 수명을 묶는다.

### Q9. 멤버 함수에서 `shared_ptr(this)`를 반환하면?

```cpp
std::shared_ptr<Player> GetSelf()
{
    return std::shared_ptr<Player>(this);
}
```

**학습자 답변**

처음 접한 문제라 설명을 요청했다.

**평가: 🔴 → 🟢**

이미 다른 `shared_ptr`가 객체를 관리하고 있어도 `shared_ptr(this)`는 새 제어 블록을 만든다. 두 제어 블록이 같은 객체를 각각 삭제해 이중 해제가 발생한다.

```cpp
class Player : public std::enable_shared_from_this<Player>
{
public:
    std::shared_ptr<Player> GetSelf()
    {
        return shared_from_this();
    }
};
```

`shared_from_this()`는 기존 제어 블록을 공유한다. 객체가 먼저 `shared_ptr`로 관리되고 있어야 하며, 생성자 안에서는 아직 연결이 완료되지 않았으므로 호출하지 않는다.

### Q10. `shared_ptr`가 스레드 안전하다는 의미는?

**학습자 답변**

처음 접한 내용이라 설명을 요청했다.

**평가: 🔴 → 🟢**

서로 다른 `shared_ptr` 복사본들이 공유하는 제어 블록의 참조 횟수 증감은 안전하다. 그러나 관리 대상 객체의 멤버 데이터는 자동으로 보호되지 않는다.

```cpp
player->score++; // 여러 스레드가 동시에 실행하면 데이터 레이스
```

필요에 따라 `std::atomic`, `std::mutex`, `std::lock_guard` 등으로 대상 데이터를 별도로 동기화해야 한다. 동일한 `shared_ptr` 변수 하나에 여러 스레드가 동시에 대입하거나 `reset()`하는 작업도 별도 동기화가 필요하다.

### Q11. 모든 동적 객체에 `shared_ptr`를 사용하면 안 되는 이유는?

**학습자 답변**

추가 자원을 사용하고, 진정한 소유자가 누구인지 알기 어려워지며, 수명이 제대로 끝나지 않는 문제가 생길 수 있다고 답했다.

**평가: 🟢**

제어 블록과 원자적 참조 횟수 연산의 비용이 있고, 파괴 시점과 수명 책임자가 불명확해지며, 순환 참조가 발생할 수 있다. 기본 선택은 값 또는 `unique_ptr`이며 실제 공동 소유가 필요할 때만 `shared_ptr`를 사용한다.

## 소유권 선택 순서

```text
값으로 충분함
→ 일반 객체

동적 수명이 필요하고 소유자가 하나
→ unique_ptr

여러 주체가 실제로 수명을 공동 책임
→ shared_ptr

shared_ptr 객체를 수명 연장 없이 관찰
→ weak_ptr

소유자가 수명을 보장하고 잠시 접근만 함
→ raw pointer 또는 reference
```

## 핵심 암기 골격

```text
unique_ptr
- 단독 소유
- move로 이전
- 기본 선택

shared_ptr
- 공동 소유
- 강한 참조 수
- 0이 되면 파괴

weak_ptr
- 비소유 관찰
- 수명 연장 안 함
- lock으로 확인
```

주의사항은 다음 두 줄로 기억한다.

```text
순환 참조 → weak_ptr로 끊는다.
제어 블록 중복 → raw pointer로 shared_ptr를 반복 생성하지 않는다.
```

## 30~60초 모범 답변

> `unique_ptr`는 하나의 객체를 한 명만 소유하는 단독 소유권을 표현합니다. 복사는 불가능하지만 `std::move`를 통해 소유권을 이전할 수 있으며, 소유자가 소멸하면 객체도 자동으로 파괴됩니다. 따라서 동적 객체의 소유자가 하나라면 기본적으로 `unique_ptr`를 사용합니다. `shared_ptr`는 여러 주체가 객체의 수명을 공동으로 책임할 때 사용합니다. 제어 블록의 강한 참조 수를 관리하며, 마지막 `shared_ptr`가 사라져 참조 수가 0이 되면 객체가 파괴됩니다. `weak_ptr`는 `shared_ptr`가 관리하는 객체를 소유하지 않고 관찰합니다. 강한 참조 수를 증가시키지 않으며 사용할 때는 `lock()`으로 객체가 살아 있는지 확인합니다. `shared_ptr` 사용 시에는 순환 참조와 같은 원시 포인터에 대한 제어 블록 중복 생성을 주의해야 합니다.

## 복습 질문

1. `unique_ptr`, `shared_ptr`, `weak_ptr`는 각각 어떤 소유 관계를 표현하나요?
2. `weak_ptr::lock()`은 무엇을 반환하며 언제 실패하나요?
3. 순환 참조를 끊을 `weak_ptr`의 방향은 어떻게 결정하나요?
4. `weak_ptr`가 `unique_ptr`를 직접 관찰할 수 없는 이유는 무엇인가요?
5. `get()`, `release()`, `reset()`은 소유권에 어떤 영향을 주나요?
6. 같은 원시 포인터로 `shared_ptr`를 두 번 만들면 왜 이중 해제가 발생하나요?
7. `shared_ptr(this)`와 `shared_from_this()`의 차이는 무엇인가요?
8. `shared_ptr`의 스레드 안전성이 관리 대상 객체까지 보호하지 않는 이유는 무엇인가요?
9. 모든 포인터를 `shared_ptr`로 만들면 안 되는 이유는 무엇인가요?

## 다음 학습

Day 09: **템플릿과 STL 이터레이터**
