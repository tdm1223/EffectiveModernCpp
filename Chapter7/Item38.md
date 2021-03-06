# 항목 38. 스레드 핸들 소멸자들의 다양한 행동 방식을 주의하라
- 합류 가능 `std::thread`는 바탕 시스템의 실행 스레드에 대응된다.
- 지연되지 않은 과제에 대한 **미래 객체**도 시스템 스레드에 대응된다.
- std::thread 객체와 미래 객체 모두 시스템 스레드에 대한 핸들이라고 할 수 있다.

## 미래 객체
- 미래 객체는 **피호출자**가 결과를 호출자에게 전송하는 **통신 채널의 한 쪽 끝**이다.
- 피 호출자는 자신의 계산 결과를 그 통신 채널에 기록한다.(보통은 `std::promise` 객체를 통해서)
- 호출자는 미래 객체를 이용해서 그 결과를 읽는다.

### 피호출자의 결과가 저장되는 위치
- 피호출자의 `std::promise`에 저장할 수는 없다.
  - 호출자가 해당 미래 객체에 대해 `get`을 호출하기 전에 피호출자의 실행이 끝날수도 있기 떄문이다.
  - `std::promise` 객체는 피호출자의 지역 범위에 있으므로 피호출자가 완료되면 함께 파괴된다.
- 호출자의 미래 객체에 저장할 수는 없다.
  - `std::future`를 이용해서 `std::shared_future`를 생성할 수 있으며 원본 `std::future`가 파괴된 후에도 `std::shared_future`를 여러 번 복사할 수 있기 때문이다.
  - 스레드의 결과가 복사를 지원하지 않는 경우 다수의 미래 객체 중 어떤 것에 결과를 저장해야 할지 알 수 없다.

## 공유 상태
- **호출자**에 연관된 객체에도, **피호출자**에 연관된 객체에도 **피호출자의 결과**를 담기는 적합하지 않다.
  - 둘의 바깥에 있는 장소에 결과를 담아야 한다.
  - 그 장소가 공유된 상태로 줄여서 공유 상태이다.
- 일반적으로 공유 상태는 힙 기반 객체로 표현되나 형식과 인터페이스, 구현은 표준이 구체적으로 명시하지 않는다.
- 표준 라이브러리 작성자는 공유 상태를 자신이 원하는 그 어떤 방식으로도 구현할 자유가 있다.
- 공유 상태의 존재가 중요한 것은 **미래 객체 소멸자의 행동**을 미래 객체와 연관된 **공유 상태가 결정**하기 때문이다.

### 공유 상태의 행동 결정 규칙
- `std::async`를 통해서 시동된 비지연 과제에 대한 공유 상태를 참조하는 마지막 미래 객체의 소멸자는 **과제가 완료될 때까지 차단**된다.
  - 미래 객체의 소멸자는 과제가 **비동기적으로 실행**되고 있는 스레드에 대해 암묵적인 `join`을 수행한다.
- 다른 모든 미래 객체의 소멸자는 그냥 해당 미래 객체를 파괴한다.
  - 비동기적으로 실행되고 있는 과제의 경우 이는 바탕스레드에 암묵적 `detach`를 수행하는 것과 비슷하다.
  - 지연된 과제를 참조하는 마지막 미래 객체의 경우 이는 그 지연된 과제가 절대로 실행되지 않음을 뜻한다.

### 정상 행동
- 정상 행동은 **미래 객체의 소멸자가 미래 객체를 파괴**한다는 것이 전부이다.
- 소멸자는 바탕 스레드를 그 무엇과도 합류시키지 않는다.(`join`)
- 그 무엇으로부터도 탈착하지 않는다.(`detach`)
- 그 무엇도 실행하지 않는다.
- 미래 객체의 자료 멤버들만 파괴한다.

### 정상 행동에 대한 예외
- 이 정상 행동에 대한 예외는 다음 조건들을 모두 만족하는 미래 객체에 대해서만 일어난다.
  - 미래 객체가 std::async 호출에 의해 생성된 공유 상태를 참조한다.
  - 과제의 시동 방침이 [std::launch::async](/Chapter7/Item36.md)이다.
  - 미래 객체가 공유 상태를 참조하는 마지막 미래 객체이다.
- 위 3가지 조건이 모두 성립할 때에만 **미래 객체의 소멸자**는 비동기적으로 실행되는 과제가 완료될 때까지 **소멸자의 실행이 차단**된다.
- 실용적인 관점에서는 `std::async`로 생성한 과제를 실행하는 스레드에 대해 암묵적 `join`을 호출하는 것에 해당한다.
- 미래 객체 소멸자의 정상 행동에 대한 이런 예외적 행동을 std::async에서 비롯된 미래 객체의 소멸자는 차단된다 라는 문장으로 요약하기도 한다.

### std::async로 시동된 비지연 과제에 대한 공유 상태에 관해 특별 규칙이 필요한 이유
- 표준 위원회는 암묵적 detach와 관련된 문제들을 피하려 했으나, 필수적인 프로그램 종료라는 급진적인 방침을 채용하는 것도 꺼렸다.
  - 결국 선택한 것이 **암묵적 join**이다.
- 미래 객체에 대한 `API`는 주어진 미래 객체가 `std::async` 호출에 의해 생긴 공유 상태를 참조하는지 판단할 수 있는 수단을 제공하지 않는다.
- 임의의 미래 객체에 대해 그 소멸자가 비동기적으로 실행되는 과제의 완료를 기다리느라 차단될 것인지를 알아내는 것은 불가능하다.
- 이로부터 몇 가지 흥미로운 결과가 빚어진다.

```cpp
std::vector<std::future<void>> futs;

class Widget{
public:
    ...
private:
    std::shared_future<double> fut;
}

```
- 주어진 미래 객체가 특별한 소멸자 행동을 유발하는 조건들을 만족하지 않음을 미리 알 수 있다면 미래 객체의 소멸자가 차단되는 일이 없을 것을 확신할 수 있다.

## std::packaged_task
- `std::packaged_task` 객체는 주어진 함수를 비동기적으로 실행할 수 있도록 포장한다.
- 포장된 함수의 실행 결과는 공유 상태에 저장된다.
- 공유 상태를 참조하는 미래 객체를 얻으려면 `get_future`함수를 호출하면 된다.

```cpp
int calcValue();                          // 실행할 함수
std::packaged_task<int()> pt(calcValue);  // 비동기적 실행을 위해 calcValue를 포장한다.
auto fut = pt.get_future();               // pt에 대한 미래 객체를 얻는다.
```
- 이 경우 미래 객체 `fut`가 `std::async` 호출로 만들어진 공유 상태를 참조하지 않음이 명확하므로 소멸자는 정상적으로 행동한다.
- 성공적으로 생성한 `std::packaged_task` 객체는 임의의 스레드에서 실행할 수 있다.
- `std::packaged_task` 객체는 복사할 수 없으므로 `pt`를 `std::thread` 생성자에 넘겨줄 때에는 반드시 [오른값으로 캐스팅](/Chapter5/Item23.md)해야 한다.

```cpp
{
    std::packaged_task<int()> pt(calcValue);
    auto fut = pt.get_future();
    std::thread t(std::move(pt));
    ...
}
```
- ... 부분에서 `t`에 대해 어떤 일이 일어나는 지는 크게 세 가지로 나눌 수 있다.
- `t`에 대해 아무 일도 일어나지 않는다.
  - 이 경우 범위의 끝에서 `t`는 합류 가능 스레드이다.
  - 프로그램이 종료된다.
- `t`에 대해 `join`을 수행한다.
  - 호출 코드에서 `join`을 수행한다면 `fut` 의 소멸자에서는 `join`을 수행할 필요가 없으며 차단될 이유가 없다.
- `t`에 대해 `detach`를 수행한다.
  - 호출 코드에서 `detach`를 수행한다면 `fut`의 소멸자에서 그것을 수행할 필요가 없다.
- `std::packaged_task`에 의해 만들어진 **공유 상태**를 참조하는 미래 객체가 있다면 소멸자의 특별한 행동을 고려한 코드를 작성할 필요가 없다.
  - 종료와 합류, 탈착에 대한 결정은 이미 해당 `std::thread`를 조작하는 코드에서 내려지기 때문이다.