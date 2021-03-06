# 항목 29. 이동 연산이 존재하지 않고, 저렴하지 않고, 적용되지 않는다고 가정하라
## 이동 의미론
- `C++11`에서 가장 주된 기능은 이동 의미론이다.
- 이동 의미론 덕분에 컴파일러는 비용이 큰 **복사 연산**을 비교적 비용이 저렴한 **이동 연산**으로 대체할 수 있다.
  - 조건이 만족되는 경우에는 반드시 대체해야 한다.
- `C++98` 코드 기반을 `C++11`을 준수하는 컴파일러와 표준 라이브러리로 컴파일하면 소프트웨어가 저절로 빠르게 실행된다.
- 성능 개선을 야기할 수 있다는 점에서 이동 의미론은 좋은 기능이라는 후광을 가질 자격이 있다.

## 이동 연산이 좋지 않다고 가정해야 하는 이유
- 이동 의미론을 **지원하지 않는 형식**들이 많다.
- `C++11`은 `C++98` 표준 라이브러리를 전체적으로 개정했는데 이동을 복사보다 빠르게 구현할 수 있는 형식들에는 이동 연산을 추가하였다.
- 라이브러리 구성요소들의 구현을 그런 연산들의 장점을 취하도록 개정했다.
- 응용 프로그램에 있는 형식들이 `C++11`에 맞게 완전히 수정되지 않았따면 컴파일러가 이동을 지원한다고 해도 응용 프로그램의 성능이 저절로 높아지지는 않을 것이다.
- 이동 연산을 명시적으로 지원하지 않는 형식에 대해 C++11이 자동으로 이동 연산들을 작성해 주긴한다.
- [형식에 복사 연산이나 이동 연산, 소멸자가 하나라도 있으면 자동 작성은 일어나지 않는다.](/Chapter3/Item17.md)
- 형식의 멤버나 부모 클래스에 이동이 비활성화되어 있으면 컴파일러의 이동 연산 작성은 일어나지 않는다.
- 이동을 명시적으로 지원하는 형식에서도 성능상의 이득이 생각만큼 크지 않을 수 있다.

## 표준 라이브러리에서의 이동과 복사
### std::vector의 이동과 복사
- `std::vector`의 내부에서는 힙 메모리를 이용하여 자료들을 저장하고 이 힙 메모리를 가리키는 **포인터**를 이용해서 동작한다.
- `std::vector`를 이동시키게 되면 포인터를 전달하면 되기 때문에 상수 시간에 이동이 끝난다.

### std::array의 이동과 복사
- `std::array`는 내장 배열에 `STL` 인터페이스를 씌워놓은 형태이다.
- 이동을 하게 되면 내장 배열에 있는 자료들을 하나씩 이동시켜야한다.
- 이동 또한 선형 시간이 필요하며, 수행 시간이 복사와 크게 차이가 나지 않는다.

### std::string
- `std::string`은 상수 시간 이동과 선형 시간 복사를 제공한다.
- `SSO(Small String Optimization)`라고 불리는 작은 문자열 최적화 때문에 이동이 빠르지 않을 수 있다.
- 작은 문자열의 경우 `std::string`은 **힙이 아닌 버퍼에 저장**한다.

## 복사가 일어나는 경우
- 빠른 이동 연산을 지원하는 형식에서도 이동이 일어날만한 상황에서 복사가 일어나는 경우가 생기기도 한다.
- **이동 연산들이 예외를 던지지 않음이 확실한 경우**에만 바탕 복사 연산들을 이동 연산들로 대체한다.
  - [표준 라이브러리의 일부 컨테이너 연산들은 강한 예외 안정성을 보장](/Chapter3/Item14.md)한다.
  - 보장에 의존하는 `C++98`코드를 `C++11`에서 컴파일해도 코드가 망가지지 않기 위해서이다.
- 이동 연산이 `noexcept`로 선언되지 않았다면 컴파일러는 여전히 복사 연산을 호출할 수도 있다.

## C++11의 이동 의미론이 도움이 되지 않는 상황 정리
### 이동 연산이 없다.
- 이동할 객체가 이동 연산들을 제공하지 않는다.
- 이동 요청은 복사 요청이 된다.

### 이동이 더 빠르지 않다.
- 이동할 객체의 이동 연산이 복사 연산보다 빠르지 않다.

### 이동을 사용할 수 없다.
- 이동이 일어나려면 이동 연산이 예외를 방출하지 않아야 하는데 해당 연산이 `noexcept`로 선언되어 있지 않다.

### 원본 객체가 왼값이다.
- [아주 드문 경우](/Chapter5/Item25.md) 오직 오른값만 이동 연산의 원본이 될 수 있는 경우도 있다.