
# 배운 Techniques
- Local Reasoning with value types
- Generic types for fast, safe polymorphism
- Composition of values

# Local Reasoning

코드를 처음 봤을때, 코드 다른 곳으로 눈을 돌릴 필요 없이 바로 앞에 있는 **local** 한 코드로만 이해할 수 있는 것.

즉, 하나의 코드를 이해하기위해 코드베이스 곳곳을 구경할 필요없는 것이 **Local Reasoning** 이다.

**Protocol** 과 **Value** 타입을 적절하게 사용하면 **Local Reasoning** 을 향상시킬 수 있다.

# class 를 struct 로 바꿔보기

**Before**
```swift
class DecoratingLayoutCell: UITableViewCell {
    var content: UIView
    var decoration: UIView
    
    //perform layout
}
```
위의 layout 로직이 Cell 내부에 갇혀있을 필요가 없음
`Struct` 로 변환

**After**
```swift
struct DecoratingLayout {
    var content: UIView
    var decoration: UIView
    
    mutating func layout(in rect: CGRect) {
       //perform layout 
    }
}
```
이렇게되면 perform layout 과 관련된 로직이 isolate 됨

그리고

```swift
class DreamCell: UITableViewCell {
    ...
    
    override func layoutSubviews() {
        var decoratingLayout = DecoratingLayout(content: content, decoration: decoration)
        decoratingLayout.layout(in: bounds)
    }
}

class DreamDetailView: UIView {
    ...
    
    
    override func layoutSubviews() {
        var decoratingLayout = DecoratingLayout(content: content, decoration: decoration)
        decoratingLayout.layout(in: bounds)
    }
}

```

## struct 를 사용하면서 생긴 변화
### 1. 낮아진 결합도 + 재사용성
아까처럼 `UITableViewCell` 을 상속받아서 **customizing** 했을때보다, 지금이 **결합도** 낮아졌기 때문에 여기저기서 재사용이 가능해졌다는 것을 볼 수 있다.

### 2. 테스팅
**단위 테스트** 를 작성하기가 편해졌다.
```swift
func testLayout() {
    let child1 = UIView()
    let childe2 = UIView()
    
    var layout = DecoratingLayout(content: child1, decoration: child2)
    layout.layout(in: CGRect(x: 0, y: 0, width: 120, height: 40))
    
    XCTAssertEqual(child1.frame, CGRect(x: 0, y: 5, width: 35, height: 30))
    XCTAssertEqual(child2.frame, CGRect(x: 35, y: 5, width: 70, height: 30))
}
```
테스트를 위해 `UITableView` 를 만들 필요도 없고,
`viewLayout` 콜백을 기다릴 필요도 없다.

### 3. Local Reasoning 향상

**struct** 를 사용하면서 더 작은 단위의 코드로 바뀌었고, 

왜냐하면 그 **struct** 내의 코드만 이해하면 되기 때문에 **Local Reasoning** 이 향상됐다고 볼 수 있다.


# Protocol 사용해보기

만약에 아래 두개의 공통점을 묶어서 하나로 사용하고싶으면 어떻게 해야될까?
공통적인 superclass 가 없어서 하나를 상속 받는 방법은 할 수가 없는 상태이다.

```swift
struct ViewDecoratingLayout {
    var content: UIView
    var decoration: UIView
    
    mutating func layout(in rect: CGRect) {
        content.frame = ...
        decoration.frame = ...
    }
}
```

```swift
//`SpriteKit` 을 사용하기 위한 layout 코드
struct NodeDecoratingLayout {
    var content: SKNode
    var decoration: SKNode
    
    mutating func layout(in rect: CGRect) {
        content.frame = ...
        decoration.frame = ...
    }
}
```
**Protocol** 을 활용하면 된다.
공통적인 기능은 `frame` 을 setting 하는거 밖에 없기 때문에 그 부분을 `protocol` 로 빼주면 된다.

```swift
protocol Layout {
    var frame: CGRect { get set }
}
```
그리고 아래에서 content와 decoration의 타입을 `Layout` 프로토콜로 바꿔준다.
```swift
struct ViewDecoratingLayout {
    var content: Layout
    var decoration: Layout
    
    mutating func layout(in rect: CGRect) {
        content.frame = ...
        decoration.frame = ...
    }
}
```
그리고 `UIView`, `SKNode` 이 `Layout` 프로토콜을 채택하게 하면 된다. (이걸 **retroactive modeling** 이라고 했는데 뭔지 아직 모르겠다 )
```swift
extension UIView: Layout {}
extension SKNode: Layout {}
```

**이렇게 다형성(polymorphism)을 superclass가 아닌 protocol로 구현하면 좋다**

## Generic 사용

```swift
struct ViewDecoratingLayout {
    var content: Layout
    var decoration: Layout
    
    mutating func layout(in rect: CGRect) {
        content.frame = ...
        decoration.frame = ...
    }
}
```
위 코드에서 `content` 에는 `UIView` 타입이 들어가고
`decoration` 에는 `SKNode` 가 들어가게 될건데

이걸 다음과같이 **Generic** 으로 표현해볼 수 있다.


```swift
struct DecoratingLayout<Child: Layout> {
    var content: Child
    var decoration: Child
    
    mutating func layout(in rect: CGRect) {
        content.frame = ...
        decoration.frame = ...
    }
}

protocol Layout {
    var frame: CGRect { get set }
}

extension UIView: Layout {}
extension SKNode: Layout {}
```
### Generic의 장점
- More control over types
- Can be optimized more at compile time
    - 코드가 어떤걸 하고있는지에 대한 정보를 컴파일러에게 더 알려주는것이기 때문에 성능이 더 좋아질 수 있다.



# Composition

**상속** 을 사용하면 **Local Reasoning** 은 저하된다.

그래서 코드를 쉐어하기에 더 좋은 방법으로 **Composition** 을 활용할 수 있다.

**Composition: Share code without reducing local reasoning**

작은 코드를 합쳐서 큰 것을 만드는것.


## Composition 은 View가 아닌 Value 타입으로 한다

왜?
- 클래스 인스턴스는 비용이 크다 (힙 영역 할당 때문에)
- 구조체가 비용이 더 낮다
- Composition 은 Value Semantics 과 더 궁합이 더 좋다
    - Value 타입은 **encapsulation**을 제공된다.

## 코드로 보자
```swift
protocol Layout {
    var frame: CGRect { get set }
}

struct CascadingLayout<Child: Layout> {
    var children: [Child]
    mutating func layout(in rect: CGRect) {
        ...
    }
}

struct DecoratingLayout<Child: Layout> {
    var content: Child
    var decoration: Child
    
    mutating func layout(in rect: CGRect) {
        content.frame = ...
        decoration.frame = ...
    }
}
```

두 종류의 `layout` 끼리 **compose** 할 수 있도록 위 코드를 더 일봔화 시켜면 
프로토콜을 다음과 같이 바꿔볼 수 있다.

```swift
protocol Layout {
    var frame: CGRect { get set }
}
```

일단 위에서 `frame` 의 `getter` 는 사용안하기 때문에 메서드로 바꿀 수 있다.
즉, 타입에서 `frame` 자체가 있고 없고는 필요없기 때문에 바꿔 볼 수 있는것이다.

```swift
protocol Layout {
    mutating func layout(int rect: CGRect)
}
```

이렇게 바꾸면 `UIView` 와 `SKNode` 처럼 
`DecoratingLayout` 과 `CascadingLayout` 도 `Layout` 을 채택할 수 있다.

```swift
extension UIView: Layout {}
extension SKNode: Layout {}

struct DecoratingLayout<Child: Layout>: Layout {}
struct CascadingLayout<Child: Layout>: Layout {}
```

그러면 이제 두 종류의 `layout` 끼리 **composing** 이 가능해졌다.

```swift
let decoration = CascadingLayout(children: accessories)
let composedLayout = DecoratingLayout(content: content, decoration: decoration)
composedLayout.layout(in: rect)
```

복잡한 타입들을 **composition** 을 활용하면 이렇게 **declarative** 하게 만들 수 있다.

### 결론
**코드를 재사용해야될때** 또는 **타입의 behaviour 을 customizing** 해야될때 **composition** 을 활용하면 좋다.


# Associated Types

```swift
protocol Layout {
    mutating func layout(int rect: CGRect)
    
    associatedType Content
    var contents: [Content] { get }
}
```
`Content` 자리에 한 타입만 들어갈 수 있도록 `associatedType` 으로 한정을 짓는다.

그럼 타입이 프로콜을 채택할때

```swift
struct ViewDecoratingLayout: Layout {
    mutating func layout(in rect: CGRect)
    typealias Content = UIView
    var contents: [Content] { get }
}

struct NodeDecoratingLayout: Layout {
    mutating func layout(in rect: CGRect)
    typealias Content = SKNode
    var contents: [Content] { get }
}
```

이렇게 사용 할 수 있다

근데 이걸 합치려면 어떠게 해야될까??

바로 **제네릭**!!

```swift
struct DecoratingLayout<Child: Layout>: Layout {
    var content: Child
    var decoration: Child
    
    mutating func layout(in rect: CGRect)
    typealias Content = Child.Content
    var contents: [Content] { get }
}
```

근데 위 코드에서 `content` 과 `decoration` 이 같은 타입인지 확인을 하고 통일을 하려면, **Generic Constraints** 를 사용하면 된다.

```swift
struct DecoratingLayout<Child: Layout, Decoration: Layout where Child.Content == Decoration.Content>: Layout {
    var content: Child
    var decoration: Child
    
    mutating func layout(in rect: CGRect)
    typealias Content = Child.Content
    var contents: [Content] { get }
}
```


## Testing

이제 단위 테스트 코드를 다음과 같이 작성할 수 있다.

```swift
func testLayout() {
    let child1 = TestLayout()
    let child2 = TestLayout()
    
    var layout = DecoratingLayout(content: child, decoration: child2)
    layout.layout(in: CGRect(x: 0, y: 0, width: 120, height: 40))
    
    XCTAssertEqual(layout.contents[0].frame, CGRect(x: 0, y: 5, width: 35, height: 30))
    XCTAssertEqual(layout.contents[1].frame, CGRect(x: 35, y: 5, width: 70, height: 30))
    
}

struct TestLayout: Layout {
    var frame: CGRect
    ...
}
```

**이러면 테스트는 **UIView** 와 분리 될 수 있다.**

# enum으로 View의 State 나타내기

뷰의 상태를 뷰컨에서 변수로 나타낼때 서로 
**mutually exclusive 하지않으면 오류가 발생할 확률이 올라간다**

이걸 **enum** 으로 구현하면 된다.
스위프트의 type system 이 강제로 case 끼리 **mutually exclusive** 된다.

```swift

enum State {
    case vieweing
    case sharing(dreams: [Dream])
    case selecting(selectedRows: IndexSet)
}
```

