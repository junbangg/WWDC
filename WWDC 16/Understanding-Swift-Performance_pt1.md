
# 스위프트의 성능

스위프트에서 성능을 고려할때 신경써야하는 것이 크게 3가지 있다.

<img width="1270" alt="Screen Shot 2021-12-23 at 11 38 42 PM" src="https://user-images.githubusercontent.com/33091784/147314658-4815025b-caba-4d3f-8c64-9ce864577d82.png">


**Allocation**

스택에 저장되는지 힙에 저장되는지

**Reference Counting** 

인스턴스를 여기저기 전달할때 발생하는 참조 카운팅에 의한 복잡도

**Method Dispatch**

인스턴스 메서드를 호출할때 **Static** 또는 **Dynamic** Dispatch 가 발생하는지

# Allocation

## Stack

스택에 메모리를 할당할때 **Stack pointer** 를 이용해서 한다.

**Stack Pointer** 는 항상 스택의 끝을 가리키며, 메서드가 호출될때, 스택 포인터를 **decrement** 해서 필요한 메모리를 할당해준다.

반대로, 호출된 메서드가 끝나면, 스택 포인터가 다시 **increment** 되면서 다시 메모리를 해제시켜준다.

### Stack Allocation 성능

스택의 메모리를 할당해주는 복잡도는 `Int` 를 변수/상수에 저장하는거와 거의 똑같다고 한다. 즉, 거의 **O(1)** 이라고 볼 수 있을것 같다.

## Heap

**Stack** 보다 더 **Dynamic** 하지만 덜 **효율적**이다

**힙의 메모리를 할당해주려면** 할당해줄 빈 공간을 탐색해야되는 비용이 발생한다.

그리고 **힙의 메모리를 해제해줄 때는** 그 메모리 공간을 다시 제자리에 꽂아넣어 줘야한다.

이러면 이미 복잡도는 **Stack** 에 비해서 더 올라가는것을 짐작할 수 있다.

그런데, 이건 시작에 불과라고 한다.

왜냐하면 멀티쓰레딩을 할때, 여러개의 쓰레드가 동시에 힙 메모리를 접근할 수 있기 때문이다.

이러면 `Lock` 과 같은 동기화 기술을 사용해야되는데, 이 역시 오버헤드를 발생시킨다.

### 코드 예시

<img width="1270" alt="Screen Shot 2021-12-24 at 12 34 52 AM" src="https://user-images.githubusercontent.com/33091784/147314668-eb392639-eab1-434b-ae2c-1efd2819bf32.png">


 (`point2.x = 5` 를 할때, `point2.y` 는 아직 0 인데.. 이걸 `value semantics` 라고 부른다는데 그게 뭔지 아직 모르겠다.)

아무튼 위 처럼 `point1` , `point2` 에 대한 스택 공간이 할당된 후에는, `stack pointer` 가 다시 메서드가 시작되기 전 상태로 **increment** 되면서 `point1`, `point2` 를 다시 `pop` 해준다.

반면에 같은 코드를 **클래스** 로 변경 했을때 **힙** 에서 어떻게 저장되는지 살펴보면,

<img width="1236" alt="Screen Shot 2021-12-24 at 8 46 30 AM" src="https://user-images.githubusercontent.com/33091784/147314675-47ba8066-1d8a-4e4c-a17d-479a4a698bcb.png">


`let point1` , `var point2` 에 `Point` 인스턴스의 주소값이 저장되기 때문에, **Stack** 의 공간도 할당이 된다.

그리고 **Heap** 은 위에서 말했던것 처럼 할당 해줄 수 있는 메모리 공간을 탐색해서 제공해줄텐데

일단 `Point()` 를 생성하면, 스위프트는 **Heap** 를 **Lock** 하는 과정을 거친다. 

이건 임계구역 문제와 연관성이 있는데, 다중 쓰레드 환경에서 생각해봤을때, 여러개의 쓰레드에서 **Heap** 

에 동시에 접근해서 같은 인스턴스 참조를 변경시킬 수 있기 때문이다.

<img width="1236" alt="Screen Shot 2021-12-24 at 8 50 00 AM" src="https://user-images.githubusercontent.com/33091784/147314698-6a033043-d543-4a14-bffd-5d0865607be3.png">


그런데 **Heap** 이 제공해준 메모리 공간이 **Stack** 과는 다르게 4개의 공간이라는 것을 볼 수 있다.

2개는 `x`, `y` 를 저장 하기 위해서이고, 나머지 두개는 스위프트에서 관리해주는 공간이라고 한다.

<img width="1236" alt="Screen Shot 2021-12-24 at 8 54 48 AM" src="https://user-images.githubusercontent.com/33091784/147314704-59e680b9-14c4-4d55-84ba-e64eabaa9c41.png">


여기서 만약에 `point1` 을 `point2` 변수에 저장을 하면 무슨 일이 일어날까?

**Stack** 에서 처럼 새로운 값이 생겨서 복사가 되는것이 아니라, `Point` 인스턴스에 대한 참조만 복사해서 `point2` 도 하게 되는 것이다.

그리고 여기서 `point2.x = 5` 를 해주면, `point1` , `point2` 가 둘다 같은 참조를 가리키고 있기 때문에, 두 곳에서 모두 `x = 5` 가 될것이다.

이걸 **Reference Semantics** 이라고 하는데, 위의 **Value Semantics** 와 대조 되는것을 볼 수 있다.

**Reference Semantics** 는 “unintended sharing of state” 를 초래하는데, 그럼 **lock** 같은 추가적인 처리를 해줘야 될 수도 있고, 코드의 복잡도도 증가하는 일이 생길 수 있다.

그리고 마지막으로 `point1` , `point2` 가 전부 사용이 끝나면, **Heap** 은 다시 **Lock** 을 한 다음에 메모리 공간을 원상복귀 시킬것이고, **Stack** 은 다시 **stack pointer** 를 이용해서 `point1` , `point2` 공간을 **pop** 할것이다.

**결론**

클래스가 **heap** 때문에 더 비용이 크다

클래스는 **Reference semantics** 때문에 **indirect storage** 와 **identity** 같은 장점이 있긴하다.

근데 그것들이 필요한게 아니라면 **Struct** 를 사용하는게 훨씬 좋을것이다

 

# Reference Counting

스위프트는 **Heap** 메모리를 어떻게 관리해줄까? 

스위프트는 **Heap** 에 있는 인스턴스를 관리할때 Reference Counting 을 이용해서 메모리에서 해제할지 판단한다.

**ARC** 를 공부했을때, 인스턴스의 참조 count가 올라갔다가 내려갔다 하면서, 0이 되었을때 메모리에서 해제된다고 배웠는데, 그 이상의 무언가가 있다고 한다.

가장 중요한것은 **Thread Safety** 오버헤드 인것 같다.

여러 **Thread** 에서 동시에 같은 참조 카운트를 변경시킬 수 있기 때문이다.

그래서 **atomically increment/decrement the reference count** 를 해야된다고 한다.

근데 아무래도 이렇게하면 여러 작업들이 많이 필요한데, 비용은 자연스럽게 올라갈것이다.

<img width="1236" alt="Screen Shot 2021-12-24 at 9 19 53 AM" src="https://user-images.githubusercontent.com/33091784/147314713-9dcbe0ac-10b7-45a6-897d-262a0825b4e4.png">


위 코드로 **ARC** 가 어떤 식으로 동작하는지 볼 수 있다.

`retain` 은 참조 카운트를 **atomically increment** 할것이고

`release` 는 참조 카운트를 **atomically decrement** 할것이다.

<img width="1236" alt="Screen Shot 2021-12-24 at 9 25 21 AM" src="https://user-images.githubusercontent.com/33091784/147314718-a38e1c8d-9c29-4ce0-bfc3-5cfda31d763a.png">


그림으로 보면 이렇다

`retain` 이 되면 인스턴스의 참조 카운트는 2가 된다.

<img width="1236" alt="Screen Shot 2021-12-24 at 9 26 31 AM" src="https://user-images.githubusercontent.com/33091784/147314728-c3f80a67-d301-4281-a6da-970188cb17e7.png">


그리고 `release` 가 되면서 참조카운트는 0이 되고, 스위프트는 비로서 **Heap** 을 잠시 **Lock** 한 뒤, 메모리에서 해제를 시킨다.

그렇다면 **Struct** 를 사용할때는 참조 카운팅이 발생안할까??

위의 **Struct** / **Stack** 예시에서는 **Heap** 을 사용하는 경우가 보이지 않았었다.

<img width="1236" alt="Screen Shot 2021-12-24 at 9 33 12 AM" src="https://user-images.githubusercontent.com/33091784/147314756-c2d332b0-f386-49a3-b06c-b99f477ec6fe.png">


이전 예시와는 다르게 여기서는 구조체가 클래스로 된 속성을 갖고있다.

여기서 `label` 은 `text` , `font` 에 대한 참조를 두개 하고있다.

그리고 `label` 을 `label2` 에 저장하게 되면, `text` , 그리고 `font` 에 대한 참조 카운트가 하나씩 더 증가할것이다.

<img width="1236" alt="Screen Shot 2021-12-24 at 9 40 33 AM" src="https://user-images.githubusercontent.com/33091784/147314758-60475bb9-b6d7-4a28-baad-c74960fd9db9.png">


여기서는 그러면 `retain` `release` 를 이런식으로 해줄것이다.

이렇게 볼 수 있듯이, reference counting에 대한 오버헤드가 두배가 될것이다.

**결론** 

클래스를 쓰면 Reference Counting에 대한 복잡도가 발생한다.

근데 마지막에서 확인했듯이, **Struct** 내부에 만약에 **Class** 에 대한 참조가 있으면, **Reference Counting** 에 대한 비용은 똑같이 발생할것이다.

# Method Dispatch

## Static Dispatch

런타임때 메서드가 호출되면, 스위프트에서 올바른 implementation 을 실행해야된다.

만약에 그 implementation 을 **Compile Time** 때 찾을 수 있으면, 그걸 **Static Dispatch** 라고 한다.

즉, 런타임때 바로 implementation 으로 가는것을 말한다.

이렇게되면 컴파일러가 이미 어떤 implementation 들이 있는지 확인할 수 있기 때문에 최적화를 더 할 수 있다.

## Dynamic Dispatch

반면에 **Dynamic Dispatch** 는 **Compile Time** 때 바로 어떤 implementation 으로 가야할지 판단을 못하고, **Dispatch Table** 을 통해 **Runtime** 때 찾게 될것이다.

중요한것은 **Dynamic Dispatch** 가 컴파일러의 시야를 가린다는 것이다. (최적화의 기희/가능성을 뺏는것)

## Static Dispatch vs Dynamic Dispatch

### Static Dispatch의 이점

<img width="1236" alt="Screen Shot 2021-12-24 at 9 57 47 AM" src="https://user-images.githubusercontent.com/33091784/147314891-f2c8bdfd-eb43-4bb3-b8bb-312b050f0469.png">

위 코드에서 `Point` 의 `draw()` 와 `drawAPoint` 둘다 **Static Dispatch** 된다.

즉, 컴파일러가 어떤 메서드들이 호출될것인지 정확히 알고있다는 것이다.

`drawAPoint()` 대신에 `point.draw()` 를 바로 해도 된다는 것을 알고있다는 뜻이다.

마찬가지로, `point.draw()` 대신에 `Point.draw` 를 바로 해도 된다는 것 또한 알고있다는 것이다.

그래서 메모리 상으로도, 스택만 이용해서 수행하고 바로 `pop` 된다는 것이다. 따로 두개의 **Static Dispatch** 에 대한 오버헤드도 필요 없고, call stack 을 만들어서 메서드들을 하나씩 호출 할 필요도 없다.

여기서만 봐도, **Static Dispatch** 가 **Dynamic Dispatch** 보다 빠르다는 것을 체감할 수 있다.

여러개의 **Static Dispatch** 가 있으면 컴파일러는 그걸 통째로 보고나서 거의 그냥 하나의 implementation 으로 축소 시킬 수 있을것인데

**Dynamic Dispatch** 가 여러개 있으면, 매번 막혀서 implemenation 을 찾아 다녀야될것이고, 오버헤드가 발생할것이다.

### Dynamic Dispatch의 이점

**Dynamic Dispatch** 가 필요한 이유는 뭘까?

말그대로 다이나믹하기 때문에 조금 더 유연성이 요구되는 작업에 필요할것 같은 짐작이 든다.

이게 쓰이는 곳 중에 하나는 바로 **Inheritance-Based Polymorphism** 이다.

<img width="1236" alt="Screen Shot 2021-12-24 at 10 22 31 AM" src="https://user-images.githubusercontent.com/33091784/147314908-1043ee65-06cd-4d28-a3b6-1abefe512d65.png">



**Dynamic Dispatch** 가 쓰이는 곳은 바로 아래쪽에 있는 `for` 문이다.

`drawables` 안에 들어있는 원소의 종류가 `Point` , `Line` 인데 

컴파일러는 **Compile Time** 때 바로 타입을 식별 할 수 없다.

즉, `d.draw()` 가 `Point` 한테 하는건지 `Line` 한테 하는건지 모른다는 이야기다.

그럼 어떤 implementation을 호출해야되는지 컴파일러는 어떻게 찾을까??

<img width="1236" alt="Screen Shot 2021-12-24 at 10 38 52 AM" src="https://user-images.githubusercontent.com/33091784/147314914-9bd498ab-8de1-4b68-931f-93557f5816c5.png">



일단 오른쪽 아래에 있는 `Line` 을 보면 알 수 있듯이, 컴파일러는 클래스에서 `type` 정보 가리키는 포인터를 저장하는 field도 추가해준다. (`type` 정보는 static 메모리에 저장 된다. (데이터 영역))

그리고 `d.draw()` 가 호출됐을때, 컴파일러는 클래스에서 가리키는 저 `type` (static memory) 으로 가서 **Virtual Method Table** 이라는 것을 생성한다.

**Virtual Method Table** 에는 각 implementation을 가리키는 `pointer` 가 있는데, 컴파일러는 그것을 이용해서 어떤 implementation 을 사용할지 결정한다.

이 동작 과정을 코드로 표현해보면 다음 과 같다.

<img width="1236" alt="Screen Shot 2021-12-24 at 12 45 54 PM" src="https://user-images.githubusercontent.com/33091784/147314920-b78ceaaa-2b4f-413e-bfbf-ed6bfcb582fb.png">


`d.type.vtable.draw(d)` 가 컴파일러가 올바른 타입의 메서드를 찾는 경로라고 볼 수 있을것 같다.

### Dynamic Dispatch 정리

- **Method Chaining** 시, 컴파일러가 **inlinement** 같은 최적화를 하기 힘들어진다.
- 단, 모든 클래스가 **Dynamic Dispatch** 가 필요한것은 아니다.
    - `final` 을 붙여주는 이유가 여기 있다.
    - 상속이 필요 없으면 `final` 을 붙여줘서, 컴파일러가 필요한 메서드들을 **Static Dispatch** 할 수 있게 해준다.
    - 컴파일러가 판단했을때 상속 같은게 필요 없어보이면, **Dynamic Dispatch** 를 **Static Dispatch** 으로 바꿔주기도 한다는것 같다;

# 정리

## Struct vs Class 를 판단할때 고민해볼것들

1. **이 인스턴스**가 **Stack** 과 **Heap** 중 어디에 저장될까?
2. **이 인스턴스**를 여기저기 전달할때, **reference counting** 은 얼마나 발생하고, 그로 인한 **오버헤드**는 얼마나 생길까?
3. **이 인스턴스**의 메서드를 호출할때 **Static Dispatch** 가 될까 **Dynamic Dispatch** 가 될까?

**불필요한 **Dynamism** 은 순수 비용이다!**
