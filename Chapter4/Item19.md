# 항목 19. 소유권 공유 자원의 관리에는 std::shared_ptr를 사용하라
## std::shared_ptr
- `std::shared_ptr`를 통해서 접근되는 객체의 수명은 공유 포인터가 **공유된 소유권** 의미론을 통해서 관리한다.
- 특정한 하나의 `std::shared_ptr`가 객체를 소유하는 것이 아니다.
- 객체를 가리키던 마지막 `std::shared_ptr`가 객체를 더이상 가리키지 않게 되면 자신이 가리키는 객체를 파괴한다.
  - 가비지 컬렉션 처럼 클라이언트는 공유 포인터가 가리키는 객체의 수명에 대해 신경 쓸 필요가 없다.
  - 소멸자처럼 객체의 파괴 시점은 결정론적이다.
- `delete`를 기본적인 자원 파괴 메커니즘으로 사용한다.

## reference count 
- `std::shared_ptr` 는 객체의 소멸시점을 관리하기 위하여 `reference count`를 사용한다.
  - `reference count` 가 0이 되면 메모리를 해제하게 된다.
- 생성자는 **참조횟수를 증가**시킨다
  - 이동 생성할 경우 참조횟수를 증가시키지 않는다.
- 소멸자는 **참조횟수를 감소**시킨다.
- 복사 할당 연산자는 증가와 감소를 모두 수행한다.
- `reference count`의 관리는 성능에 영향을 미친다.

### reference count가 성능에 미치는 영향
- `std::shared_ptr`의 크기는 **raw 포인터의 두 배**이다.
  - `raw` 포인터와 `reference count`를 가리키는 `raw` 포인터를 갖고 있어야 하기 때문에 `raw` 포인터의 두 배의 크기를 갖고 있다.
- 참조 횟수를 담을 메모리를 반드시 **동적으로 할당**해야 한다.
  - `reference count`를 가리키는 객체는 공유되는 자원이기 때문에 동적으로 할당해야 한다.
  - [std::make_shared](/Chapter4/Item21.md)를 이용해서 `std::shared_ptr`를 생성하면 동적 할당의 비용을 피할 수 있다.
- 참조 횟수의 증가와 감소가 반드시 **원자적 연산**이어야 한다.
  - 여러 스레드가 `reference count`를 동시에 읽고 쓰려 할 수 있기 때문이다.
  - 대체로 원자적 연산은 비원자적 연산보다 느리므로 성능 감소가 있다.

## std::shared_ptr의 커스텀 삭제자
- [std::unque_ptr](/Chapter4/Item18.md) 와 같이 `std::shared_ptr`에서도 커스텀 삭제자를 구현 할 수 있다.
- `std::unique_ptr`에서는 삭제자의 형식이 **스마트 포인터의 형식의 일부**였지만 `std::shared_ptr`에서는 그렇지 않다.

```cpp
// 커스텀 삭제자
auto loggingDel = [](Widget *pw)
                  {
                    makeLogEntry(pw);
                    delete pw;
                  };
                  
std::unique_ptr<Widget, decltype(loggingDel)> upw(new Widget, loggingDel); // 삭제자의 형식이 포인터 형식의 일부
std::shared_ptr<Widget> spw(new Widget, loggingDel);                       // 삭제자의 형식이 포인터 형식의 일부가 아님
```
- `std::shared_ptr`는 **커스텀 삭제자의 타입**을 지정해주지 않아도 된다.
  - 설계가 더 유연하다.

```cpp
auto customDeleter1 = [](Widget *pw) { ... }; // 커스텀 삭제자 1
auto customDeleter2 = [](Widget *pw) { ... }; // 커스텀 삭제자 2

std::shared_ptr<Widget> pw1(new Widget, customDeleter1);
std::shared_ptr<Widget> pw2(new Widget, customDeleter2);

std::vector<std::shared_ptr<Widget>> vpw{ pw1, pw2 }; // Widget 형식의 객체를 담는 벡터에 담을 수 있음
```
- 삭제자가 다르지만 `pw1`과 `pw2`는 같은 형식이므로 그 형식의 객체들을 담는 컨테이너 안에 집어넣을 수 있다.
- 하나를 다른 하나에 배정할 수 있다.
- `pw1`, `pw2` 둘 다 `std::shared_ptr<Widget>`형식의 매개변수를 받는 함수에 넘겨줄 수 있다.
- 커스텀 삭제자를 지정해도 `std::shared_ptr` 객체의 크기가 변하지 않는다.
  - 삭제자와 무관하게 `std::shared_ptr` 객체의 크기는 항상 포인터 두 개 분량이다.

## Control Block (제어 블록)
- `std::shared_ptr`이 가지고 있는 두 개의 포인터 중 하나가 가리키는 객체이다.
- 제어 블록에는 참조 횟수, [약한 횟수](/Chapter4/Item21.md), 커스텀 삭제자, 할당자 등이 포함된다.
- `std::shared_ptr`가 관리하는 **객체당 하나의 제어 블록**이 존재한다.
- 객체의 제어 블록은 그 객체를 가리키는 최초의 `std::shared_ptr`가 생성될때 설정된다.
- 제어 블록의 존재 여부를 알 수는 없지만 제어 블록 생성 여부에 관해 다음 규칙들을 유추할 수 있다.

### 제어 블록 생성 여부에 관한 규칙
- [std::make_shared](/Chapter4/Item21.md)는 **항상 제어 블록을 생성**한다.
  - 항상 새로운 `std::shared_ptr` 객체를 생성하기 때문에 제어 블록이 존재할 가능성이 없다.
- 고유 소유권 포인터로(`std::unique_ptr` or `std::auto_ptr`)부터 `std::shared_ptr` 객체를 생성하면 제어 블록이 생성된다.
  - 고유 소유권 포인터는 제어 블록을 사용하지 않기 때문에 제어 블록이 존재할 가능성이 없다.
- `raw` 포인터로 `std::shared_ptr` 생성자를 호출하면 제어 블록이 생성된다.
  - `raw` 포인터는 제어 블록이 없을것이라 가정하여 제어 블록을 생성한다.
  - 생성자로 `std::shared_ptr` 나 [std::weak_ptr](/Chapter4/Item20.md)를 사용하면 제어 블록을 만들지 않는다.

## std::shared_ptr 사용시 주의할 점
- `std::shared_ptr` 생성자에 `raw` 포인터를 넘겨주는 일은 피해야 한다.
  - **std::make_shared**를 사용한다.
  - **커스텀 삭제자**를 사용한다.
- `std::shared_ptr` 생성자를 `raw` 포인터로 호출할 수 밖에 없는 상황이라면 `raw` 포인터 변수를 거치지 않는다.
  - `new`의 결과를 **직접 전달**한다.

```cpp
auto pw = new Widget;
std::shared_ptr<Widget> spw1(pw, loggingDel);         // raw 포인터 전달 pw*에 대한 제어 블록 생성
std::shared_ptr<Widget> spw2(pw, loggingDel);         // raw 포인터 전달 pw*에 대한 두 번째 제어 블록 생성

std::shared_ptr<Widget> spw1(new Widget, loggingDel); // new 직접 사용
std::shared_ptr<Widget> spw2(spw1);                   // spw1, spw2는 동일한 제어 블록을 사용한다.
```
- `std::shared_ptr`의 `API`는 **단일 객체를 가리키는 포인터**만 염두에 두고 설계되었다.
  - `std::shared_ptr<T[]>`같은 것은 존재하지 않는다.