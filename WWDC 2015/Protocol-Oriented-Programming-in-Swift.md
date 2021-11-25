# WWDC 2015/ Protocol Oriented Programming in Swift

## 타입(클래스, 구조체, 열거형)이 좋은 점

- Encapsulation
- Access Control
    - 코드의 내부와 외부를 분리
- Abstraction
- Namespace
    - 소프트웨어가 커지면서 일어나는 충돌 방지
- Expressive Syntax
- Extensibility
- **위의 것들이 complexity 를 관리할 수 있게 해줌(소프트웨어 개발의 중요한 문제)**

### 클래스의 장점

- **상속**을 통해, 부모클래스의 기능을 그대로 자식 클래스가 사용할 수 있는 **Sharing** 을 제공
- 이러는 동시에, 자식 클래스에서 부모클래스의 기능을 확장시킬 수 있는 (`override` ) **Flexibility** 를 제공

### 클래스의 단점

1. **Implicit Sharing**
    - A와 B 가 공유하는 데이터의 변경이 초래할 수 있는 문제점들( **임계영역** 문제)
    - 이러면  해결하기위해 **Lock** 같은 방법을 사용하는데, 그러면 코드가 점점 비효율적으로 변함

<img width="1132" alt="Screen Shot 2021-11-13 at 5 20 40 PM" src="https://user-images.githubusercontent.com/33091784/141953876-df962cff-d750-438f-9f87-bfcafb4ac587.png">



- **이래서 스위프트의 Collection 타입들은 전부 값 타입이다**
- "The one you're iterating and the one you're modifying are distinct"
    - 값 타입이니까
    
1. **Inheritance all up in your business- too intrusive**
    - Superclass는 단 하나밖에 못 쓴다.(대충 왜 프로토콜이 짱인지 말하려고 하는듯)
    - 상속을하면서 무거워진다. 다 받아와야하기 때문에
    - 선언할때 어떤 클래스를 상속 받을지 바로 결정해야된다
    - 부모 클래스가 저장 속성이 있을지도 모른다.
        - 그러면 다 받아오고
        - 다 저장하고 initialize 해야한다
    - 뭐를 `override` 해야될지, 그리고 언제 하면 안되는지 판단해야된다.

**이래서 Delegate Pattern 사용**

1. **Lost Type Relationships**
    - 비교 연산할때 문제가 있다
    
    ```swift
    class Ordered {
        func precedes(other: Ordered) -> Bool { fatalError("implement 해주세요") }
    }
    
    func binarySearch(sortedKeys: [Ordered], forKey k: Ordered) -> Int {
        var lo = 0, hi = sortedKeys.count
        while hi > lo {
            let mid = lo + (hi - lo) / 2
            if sortedKeys[mid].precedes(k) {lo - mid + 1}
            else {hi = mid}
        }
        return lo
    }
    ```
    
- 비교를 할때 비교가능한 타입을 만들어줘야한다 (`Equatable`, `Comparable` )
    
    ```swift
    class Number: Ordered {
        var value: Double = 0
        override func precedes(other: Ordered) -> Bool {
            return value < other.value
        }
    }
    ```
    
- 이렇게 비교 가능한 타입을 만드려고 해도, `other` 는 `value` 가 있는지 없는지 알 수가 없다.
- 그래서 `downcast` 를 해줘야한다.
    - `(other as! Number).value`
- **결론은 더 좋은 Abstraction mechanism 이 필요하다**

지금까지의 내용은 `Protocol` 이 필요한 이유들이었이다.

## Protocol

**스위프트는 OOP도 가능하지만 POP로 다자인 됐다**

**"클래스로 시작하지말고 프로토콜로 시작해라" - 스위프트 띵언**

위의 `Ordered` 코드를 `protocol` 로 바꿔보면, 오류가 몇개 발생한다.

그걸 보면 클래스와 프로토콜의 차이를 몇개 확인할 수 있다.

- 포토코콜 메서드는 body가 있으면 안된다고 오류가 발생한다.
    
    "Which is actually pretty good, because it means that we're going to trade that dynamic runtime check (클래스는 힙영역에 저장되므로 런타임때 결정된다) for a static check, that precedes as implemented"
    
- 코드는 아래와 같이 바뀐다.

```swift
protocol Ordered {
    func precedes(other: Ordered) -> Bool
}

struct Number: Ordered {
    var value: Double = 0
    func precedes(other: Ordered) -> Bool {
        return value < (other as! Number).value
    }
}
```

- 메서드가 `override` 를 안하고있다(당연..클래스가 아니니까 이제)
- 지금의 `protocol` 이 예전에 의도했던 클래스가 해주는 역할과 똑같은것을 확인할 수 있다.
- 확실히 개선이 됐다. 메서드를 위에서 구현할 필요가 없으니.
- 이제 저 강제 다운캐스트만 해결하면 된다.

```swift
protocol Ordered {
    func precedes(other: Self) -> Bool
}

struct Number: Ordered {
    var value: Double = 0
    func precedes(other: Number) -> Bool {
        return value < other.value
    }
}
```

- 그러면 이렇게 바뀌는데, `other: Self` 는 타입이 자기 자신이라는 뜻이다.
- 그러면 이진탐색 코드도 바꿔야한다

```swift
func binarySearch(sortedKeys: [Ordered], forKey k: Ordered) -> Int {
    var lo = 0, hi = sortedKeys.count
    while hi > lo {
        let mid = lo + (hi - lo) / 2
        if sortedKeys[mid].precedes(k) {lo - mid + 1}
        else {hi = mid}
    }
    return lo
}
```

여기서 지금

"Protocol 'Ordered' can only be used as a generic constraint because it has Self or associated type requirements"

이런 오류가 발생하는데, 제네릭으로 바꿔달라고 하고있다.

```swift
func binarySearch<T: Ordered>(sortedKeys: [T], forKey k: T) -> Int {
    var lo = 0, hi = sortedKeys.count
    while hi > lo {
        let mid = lo + (hi - lo) / 2
        if sortedKeys[mid].precedes(other: k) {lo - mid + 1}
        else {hi = mid}
    }
    return lo
}
```

"homogeneous array" 를 받도록 한것이다. (`Ordered` 를 채택하는 다양한 타입으로 구성된 array)

<img width="1172" alt="Screen Shot 2021-11-13 at 5 59 28 PM" src="https://user-images.githubusercontent.com/33091784/141953905-c7a07d12-6b2f-4a57-b8d6-2cbbc7238ff2.png">


- dynamic dispatch vs static dispatch 는 polymorphism 이야기이다.

## POP 의 중요성

- **테스팅을 위한 POP**
    - **"The more we decouple things with protocols, the more testable everything gets"**
    - `protocol` 로 객체 간의 **결합도**를 낮추면, Testability 는 높아진다.
    - `protocol` 를 사용하면, 테스트를 위한 `mock` 의 필요성이 없어진다.
        - Mock를 사용하는것과 비슷하지만, 훨씬 좋다.
        - mock는 inherently fragile 하다.
        - 왜냐하면 테스트를 할때 테스트 코드와 실제코드의 **결합도**가 높아진다
        - Mock는 스위프트의 strong static type system과 맞지않다.
- `protocol` 의 `shared implementation` 를 이용하면 코드의 확장성이 높아진다.
    - `protocol` 에 새로운 기능을 추가해야될때, `protocol` 의 메서드를 넣어주고, 해당 프로토콜을 채택하는 모든 모델을 업데이트 시킬 필요 없이
    - `extension [protocol name]` 을 이용해서 필요한 메서드를 구현하면, 모든 모델은 이 메서드가 생긴다.
- 필요한 기능을 클래스 대신에 프로토콜로 구현해놓으면, 클래스 처럼도 사용할 수 있고(다른 타입에 채택시키면서), 확장성도 증진 됨.

## 스위프트 standard library에 적용된 POP

- `indexOf` 메서드

```swift
extension CollectionType {
    public func indexOf(element: Generator.Element) -> Index? {
        for i in self.indices {
            if self[i] == element {
                return i
            }
        }
    }
    return nil
}
```

위 코드에서 문제는, `element` 는 `==` 가 안된다. 

"binary operator `==` cannot be applied to two Generator.Element operands"

여기서 사용되는게  **constrained extension** 이다

```swift
extension CollectionType where Genrator.Element: Equatable {
    public func indexOf(element: Generator.Element) -> Index? {
        for i in self.indices {
            if self[i] == element {
                return i
            }
        }
    }
    return nil
}
```

`Generator.Element` 가 `Equatable` 를 채택할때만 이 `extension` 이 사용 가능하다

위의 이진탐색 코드에서도 똑같이 `protocol` `extension` 를 활용할 수 있다.

```swift
protocol Ordered {
    func precedes(other: Self) -> Bool
}

func binarySearch<T: Ordered>(sortedKeys: [T], forKey k: T) -> Int {...}
```

이런 상태에서, `Int`, `String` 를 넣고싶을때마다

```swift
extension Int: Ordered {
    func precedes(other: Int) -> Bool {
        return self < other
    }
}

extension String: Ordered {
    func precedes(other: String) -> Bool {
        return self < other
    }
}
```

이런식으로 `Ordered` 를 채택 해야된다.

그런데 아예 이미 `Int` 와 `String` 이 채택하고있는 `Comparable` 에 `extension` 를 사용하면

```swift
extension Comparable {
    func precedes(other: Self) -> Bool { return self < other }
}

extension Int: Ordered {}
extension String: Ordered {}
```

이런식으로 개선 시킬 수 있다.

여기서 한단계 더 개선을 시키면 `constrained extension` 를 활용해 볼 수 있다.

```swift
extension Ordered where Self: Comparable {
    func precedes(other: Self) -> Bool { return self < other }
}
```

이렇게 하면 `Comparable` 를 택하는 타입에 한해서만 `Ordered` 의 `precedes` 를 사용할 수 있게 된다.

이러면 다 따로따로 구현해줄 필요가 없다.

The same logical abstraction, coming from two different places. And we've made them interoperate seamlessly.

## Generic 은 짱이다

Swift1 에서 이랬던 기괴한 코드가

```swift
func binarySearch<
    C: CollectionType where C.index == RandomAccessIndexType,
    C.Generator.Element: Ordered
>(sortedKeys: C, forKey k: C.Generator.Element) -> Int {...}

let pos = binarySearch([2, 3, 5, 7, 11, 13, 17], forKey: 5)
```

아래와 같이 개선될 수 있었다.

```swift
extension CollectionType where Index == RandomAccessIndexType,
Generator.Element: Ordered {
    func binarySearch(forKey: Generator.Element) -> Int {
    ...
    }
}

let pos = [2, 3, 5, 7, 11, 13, 17].binarySearch(5)
```

# When to use Classes

- implicit sharing이 필요할때
- 인스터스를 복사하거나 비교할 필요가 없을때(e.g. Window)
    - 복사체가 무슨 의미인지 모르겠는데? 라는 생각이 들면 참조를 하는게 맞을 수도 있다
- 인스턴스의 라이프타임이 외부 효과와 연관될때
    - e.g. disk에 파일이 생기는 일
- 인스턴스가 `sink` 만 할때.
    - 
- Be circumspect
- 프로그램의 객체는 너무 커지면 안된다.
- refactor/factor 를 할때 value 타입으로 하는걸 고려해보자

# Summary

- `protocol` > `Superclasses`
- `protocol extension` == 마술
