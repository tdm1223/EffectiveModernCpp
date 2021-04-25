# 항목 20. std::shared_ptr처럼 작동하되 대상을 잃을 수도 있는 포인터가 필요하면 std::weak_ptr를 사용하라
## std::weak_ptr
- [std::shared_ptr](/Chapter4/Item19.md)처럼 행동하지만 **객체의 참조 횟수에는 영향을 미치지 않는 포인터**가 필요한 경우도 있다.
  - 자신이 가리키는 대상이 **이미 파괴**되었을 수도 있다는 `std::shared_ptr`의 문제점을 극복할 수 있어야 한다.
  - 자신이 가리키는 **객체가 더 이상 존재하지 않는 상황**을 검출할 수 있어야 한다.
- 표준 라이브러리에서는 스마트 포인터 `std::weak_ptr`를 제공한다.
- `std::weak_ptr`는 역참조할 수 없으며 널인지 판정할 수도 없다.
- `std::weak_ptr`는 독립적인 스마트 포인터가 아니다.
  - `std::shared_ptr`를 보강하는 포인터이다.
- 효율성 면에서 `std::weak_ptr`는` std::shared_ptr`와 동일하다.
  - 객체 크기도 같고 제어 블록을 사용하며 생성, 파괴, 할당 같은 연산에 원자적 참조 횟수 조작이 관여한다.
```cpp
auto spw = std::make_shared<Widget>();  // Widget의 참조 횟수는 1이다.
std::weak_ptr<Widget> wpw(spw);         // Widget의 참조 횟수는 여전히 1이다.
spw = nullptr;                          // Widget의 참조 횟수가 0이 되고 파괴된다.
```
- 대상을 잃은 `std::weak_ptr`를 **만료되었다**라고 말한다.
- 만료 여부는 `expired()`함수를 통해 판정할 수 있다.
```cpp
if (wpw.expired()) // 만료 여부 판정
```
- `std::weak_ptr`의 만료를 검사하여 만료되지 않았으면 피지칭 객체에 접근하는 방식의 코드가 가능하지 않을까 ?
  - `std::weak_ptr`에는 역참조 연산이 없으므로 작성 불가능하다.
- 제대로된 용법은 의 만료 여부를 점검하고 만료되지 않았으면 피지칭 객체에 대한 접근을 돌려주는 연산을 하나의 원자적 연산으로 수행하는 것이다.
  - `std::weak_ptr`로부터 `std::shared_ptr`를 생성하는데 생성 했을 때의 행동 방식에 따라 두 가지로 나뉜다.

### std::weak_ptr::lock을 사용한 방법
```cpp
std::shared_ptr<Widget> spw = wpw.lock(); // wpw가 만료이면 spw는 널
auto spw = wpw.lock();                    // 위와 동일하나 auto를 사용한 방법
```

### std::shared_ptr 생성자를 사용하는 것
- `std::weak_ptr`를 인수로 받는 `std::shared_ptr` 생성자를 사용한다.
```cpp
std::shared_ptr<Widget> spw(wpw); // wpw가 만료이면 std::bac_weak_ptr가 발생
```
- `std::weak_ptr`가 만료되었다면 예외가 발생한다.
 
## 사용하는 상황
### 캐시를 사용하는 경우
- 팩토리 함수가 주어진 고유 ID에 해당하는 **읽기 전용 객체를 가리키는 스마트 포인터**를 돌려준다고 가정한다.
  - 팩토리 함수의 반환 형식은 [항목 18](/Chapter4/Item18.md)을 참고하여 `std::unique_ptr`를 사용할 것이다.
```cpp
std::unique_ptr<const Widget> loadWidget(WidgetID id);
```
- `loadWidget`의 비용이 크고 `ID`들이 되풀이해서 쓰이는 경우가 많다면 **캐싱 하는 함수**를 작성하려 할 것이다.
- 요청이 온 모든 객체를 캐싱한다면 성능상의 문제가 발생할 것이므로 **사용하지 않는 객체는 소멸**시켜야 한다.
- 캐시에 저장할 포인터는 자신이 대상을 잃었음을 감지 할 수 있는 `std::weak_ptr`이어야 한다.
- 팩토리 함수의 반환 형식이 `std::unique_ptr`로 두는것은 바람직하지 않고 `std::shared_ptr`이어야 한다.
  - `std::weak_ptr`는 객체의 수명을 `std::shared_ptr`로 관리하는 경우에만 자신이 대상을 잃었음을 감지할 수 있기 때문이다.

```cpp
std::shared_ptr<const Widget> fastLoadWidget(WidgetID id)
{
    static std::unordered_map<WidgetID, std::weak_ptr<const Widget>> cache;
    auto objPtr = cache[id].lock(); // 캐시에 있는 객체를 가리키는 std::shared_ptr
    
    // 캐시에 없으면 적재하고 캐시에 저장
    if (!objPtr)
    {
        objPtr = loadWidget(id);
        cache[id] = objPtr;
    }
    return objPtr;
}
 ```

### 옵저버 패턴
- 옵저버 패턴의 주된 구성 요소는 관찰 대상(`subject`; 상태가 변할 수 있는 객체)과 관찰자(`observer`; 상태 변화를 통지받는 객체)이다.
- 각 관찰 대상에는 자신의 관찰자들을 가리키는 포인터들을 담은 자료 멤버가 있다.
  - 상태 변화를 관찰자들에게 손쉽게 통지할 수 있다.
- 관찰 대상은 관찰자들의 파괴 시점에는 관심이 없지만 자신이 파괴된 관찰자에 접근하는 일이 없도록 보장하는데에는 관심이 많다.
- 합당한 설계중 하나는 관찰자들을 가리키는 `std::weak_ptr` 컨테이너를 자료 멤버로 두는 것이다.
  - 만료 여부를 보고 관찰자가 유효한지 점검 후에 관찰자에 접근할 수 있다.
 
### 순환 고리 방지
- 스마트 포인터를 사용하는 경우에 발생하는 가장 큰 문제이며 `std::weak_ptr`를 사용하는 가장 큰 이유이다.
- 객체 A, B, C로 이루어진 자료구조에서 A와 C가 B의 소유권을 공유하여 B를 가리키는 `std::shared_ptr`를 가지고 있는 상황을 가정한다.
![weak1](/Img/weak_ptr1.jpg)

- 이 상황 속에서 B가 A를 가리키는 포인터가 필요하게 된 경우 어떤 포인터를 사용해야 할까 ?
![weak2](/Img/weak_ptr2.jpg)

1. 일반적인 포인터
   - A가 소멸된다면 B는 소멸되었는지 알 수가 없어 **널 포인터**에 접근하게 된다.
2. std::shared_ptr
   - A가 B를 소유하고 B가 A를 소유하게 된다.
   - **순환 참조**를 하게 되는 상황으로 서로 A와 B 모두 파괴되지 못한다.
3. std::weak_ptr
   - A가 파괴되면 A를 가리키는 B의 포인터가 대상을 잃지만 B는 그 사실을 알 수 있다.
   - A와 B가 서로를 가리키지만 B의 포인터는 A의 참조 횟수에 영향을 미치지 않는다.
   - `std::shared_ptr`들이 더 이상 A를 가리키지 않게 되면서 A가 정상적으로 파괴된다.
   - 널 포인터에 접근하지도 순환참조가 발생하지도 않는다.