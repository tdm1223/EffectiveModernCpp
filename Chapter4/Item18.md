# 항목 18. 소유권 독점 자원의 관리에는 std::unique_ptr를 사용하라
## std::unique_ptr
- 일반적인 포인터와 같은 크기라고 가정해도 된다.
- 대부분의 연산에서 일반적인 포인터와 동일한 명령들을 실행한다.
- 복사를 허용하지 않으며 **이동만 가능**하다.
- 널이 아닌 `std::unique_ptr`는 항상 **자신이 가리키는 객체를 소유**한다.
- `std::unique_ptr`를 이동하면 소유권이 원본 포인터에서 대상 포인터로 옮겨진다.
- 파괴될 때 가리키는 자원이 함께 파괴된다.

## std::unique_ptr의 활용
- `std::unique_ptr`의 흔한 용도 중 하나는 계통구조 안의 객체를 생성하는 팩토리 함수의 **반환 형식**으로 사용하는 것이다.
- 계통구조에 대한 팩토리 함수는 **힙에 객체를 생성**하고 **객체를 가리키는 포인터**를 돌려준다.
- 객체를 삭제하는 것은 호출자의 몫인데 `std::unique_ptr`는 소멸 시 피지칭 객체를 **자동으로 삭제**해줘서 적합하다.

## std::unique_ptr의 커스텀 삭제자
- 객체의 삭제는 `delete`를 통해서 일어나지만 `std::unique_ptr` 객체를 생성할 때 커스텀 삭제자를 사용하도록 지정하는 것도 가능하다.
- 커스텀 삭제자는 해당 자원의 **파괴 시점에서 호출되는 임의의 함수**이다.
- `makeInvest`가 생성한 객체를 `delete`로 파괴할 수 없고 로그 항목을 기록한 후에 파괴해야 한다면 아래처럼 구현하면 된다.
```cpp
// 커스텀 삭제자
auto delInvmt = [](Investment* pInvestment)     
                {
                    makeLogEntry(pInvestment);
                    delete pInvestment;
                };
                
template<typename... Ts>
std::unique_ptr<Investment, decltype(delInvmt)> // 반환 형식이 바뀌었음
makeInvestment(Ts&&... params)
{
    std::unique_ptr<Investment, decltype(delInvmt)> pInv(nullptr, delInvmt); // 돌려줄 포인터
      
    if ( /* Stock 객체를 생성해야 하는 경우 */ )
    {
        pInv.reset(new stock(std::forward<Ts>(params)...));
    }
    else if ( /* Bond 객체를 생성해야 하는 경우 */ )
    {
        pInv.reset(new Bond(std::forward<Ts>(params)...));
    }
    else if ( /* RealEstate 객체를 생성해야 하는 경우 */ )
    {
        pInv.reset(new RealEstate(std::forward<Ts>(params)...));
    }
    return pInv;
}
```
- 커스텀 삭제자는 `std::unique_ptr`의 **두번째 인자**로 지정하면 된다.
- `new` 호출에서는 `makeInvestment` 함수에 전달된 인수들을 `new`에 **완벽하게 전달**하기 위해 [std::forward](/Chapter5/Item25.md)를 사용하였다.
  - 호출자가 함수에 제공한 모든 정보를 함수 안에서 생성할 객체의 생성자에게 손실 없이 넘겨줄 수 있다.
- 프로그램의 모든 경로에서 파괴가 정확히 한 번만 일어나는 것이 보장된다.
- 자원의 파괴 방식에 대해 신경 쓸 필요가 전혀 없다.
- 기본 클래스의 포인터를 통해서 파생 클래스의 객체를 삭제한다.
  - 기본 클래스인 `Investment`의 소멸자가 **가상 소멸자**여야 한다.

### C++14에서 커스텀 삭제자
- `C++14`는 [함수 반환 형식의 연역을 지원](/Chapter1/Item3.md)하므로 간결하고 캡슐화된 방식으로 구현할 수 있다.
```cpp
template<typename... Ts>
auto makeInvsetment(Ts&&... params)
{
    // makeInvestment 내부에서 삭제자 정의 가능
    auto delInvmt = [](Invsetment* pInvestment)
    {
        makeLogEntry(pInvestment);
        delete pInvestment;
    };
    
    std::unique_ptr<Investment, decltype(delInvmt)> pInv(nullptr, delInvmt);
      
    if ( /* Stock 객체를 생성해야 하는 경우 */ )
    {
        pInv.reset(new stock(std::forward<Ts>(params)...));
    }
    else if ( /* Bond 객체를 생성해야 하는 경우 */ )
    {
        pInv.reset(new Bond(std::forward<Ts>(params)...));
    }
    else if ( /* RealEstate 객체를 생성해야 하는 경우 */ )
    {
        pInv.reset(new RealEstate(std::forward<Ts>(params)...));
    }
    
    return pInv;
}
```

##  커스텀 삭제자 사용시 std::unique_ptr의 크기
- 커스텀 삭제자를 사용하는 경우에는 `std::unique_ptr`의 크기가 1워드에서 2워드로 증가한다.
- 삭제자가 함수 객체일 때에는 그 **함수 객체에 저장된 상태의 크기만큼 증가**하고 람다식의 경우에는 크기의 변화가 없다.
  - 두 가지 모두 구현 가능하다면 람다식을 사용하는 것이 바람직하다.

### 람다식으로 작성한 경우
```cpp
// 상태 없는 람다 형태의 삭제자
auto delInvmt = [](Investment* pInvestment)
                {
                    makeLogEntry(pInvestment);
                    delete pInvestment;
                };
                
template<typename... Ts>
std::unique_ptr<Investment, decltype(delInvmt)>  // 반환 형식은 Investment* 와 같은 크기
makeInvsetment(Ts&&... params);
```

### 함수 형태로 작성한 경우
```cpp
// 함수 형태의 삭제자
void delInvmt2(Investment* pInvestment)
{
    makeLogEntry(pInvestment);
    delete pInvestment;
};

template<typename... Ts>
std::unique_ptr<Investment, void (*)(Investment(*)>  // 반환 형식은 Investment* 크기에 함수 포인터의 크기를 더한 크기
makeInvsetment(Ts&&... params);
```