# WWDC22 - Async/Await

# Concurrency in Swift

- 가독성
- 안정성

## Functions: Synchronous and asynchronous

**Synchronous: Thread is blocked, waiting for function to finish**

**Asynchronous**: Frees thread to do other things, completionHandler is used to notify end of function.

# Example: Fetching a Thumbnail
<img width="1164" alt="Screen Shot 2022-06-23 at 4 42 08 PM" src="https://user-images.githubusercontent.com/33091784/175307777-94366493-351c-476d-af9a-12d3095da82e.png">

여기서 각 메서드는 직전 메서드 결과에 의존한다. 즉, 직전의 결과값을 이용해서 호출된다.

<img width="1164" alt="Screen Shot 2022-06-23 at 4 43 46 PM" src="https://user-images.githubusercontent.com/33091784/175307814-1b474677-c6ec-467b-9f62-a46d1ec2fe52.png">

얘네들은 synchronous 해도 된다. 

<img width="1164" alt="Screen Shot 2022-06-23 at 4 44 23 PM" src="https://user-images.githubusercontent.com/33091784/175307858-8ba66175-b6cc-4f56-a25e-5ca404ad3bf9.png">


얘네들은 asynchronous.(시간이 걸리기 때문에)

# Before Async/await

<img width="1185" alt="Screen Shot 2022-06-23 at 4 47 08 PM" src="https://user-images.githubusercontent.com/33091784/175307889-85c6d4a9-9cd1-4332-a205-7e0df520a701.png">

<img width="1312" alt="Screen Shot 2022-06-23 at 4 47 35 PM" src="https://user-images.githubusercontent.com/33091784/175307921-c5e1907c-eca7-42de-b694-9f2401805e92.png">

<img width="1312" alt="Screen Shot 2022-06-23 at 4 48 44 PM" src="https://user-images.githubusercontent.com/33091784/175307940-91a0b5a7-d324-4feb-953f-3351ff79359c.png">

여기서 첫번째 문제?

`return` 만 하고 `error` 를 전달하지않았다

이대로라면 `UI` 에 `spinner` 만 계속 돌게될거다

이걸 수정하면 다음과 같다

<img width="1164" alt="Screen Shot 2022-06-23 at 4 59 35 PM" src="https://user-images.githubusercontent.com/33091784/175307996-899551bb-c745-4143-af81-e6fedc72f24b.png">

근데 이건 스위프트가 잘못됐다고 알려주질 않는다, 즉 **컴파일 에러** 가 발생 하지않는다

대안으로는 `Result` 를 사용하는 것

<img width="1232" alt="Screen Shot 2022-06-23 at 5 03 32 PM" src="https://user-images.githubusercontent.com/33091784/175308023-19b0fa21-59a7-401d-95f7-1921c46dbee8.png">

- 코드가 근데 길어지고 지저분해진다

# After Async/await

위 코드를 `Async/await` 이용해서 바꾸면 다음과 같다

```swift
// completionHandler 대신 그냥 async 사용
// throws 는 그대로 에러를 던질 수 있다는 의미
func fetchThumbnail(for id: String) async throws -> UIImage {
	let request = thumbnailURLRequest(for: id)
	// 위에 throws로 표시되어있기 때문에 try 도 있다
	// 이전 코드 처럼 오류 체킹 후에 completionHandler 를 통해 전달하는 코드가 try 로 간소화 됨
  // 마찬가지로 위에 async가 있기 때문에 여기 await 도 있다
	let (data, response) = try await URLSesesion.shared.data(for: request)

	guard (response as? HTTPURLResponse)?.statusCode == 200 else {
		throw FetchError.badID
	}
	let maybeImage = UIIMage(data: data)
	guard let thumbnail = await maybeImage?.thumbnail else {
		throw FetchError.badImage
	}
	return thumbnail
}
```

이렇게 작성하는 장점은, 위에서는 없었던 컴파일 에러가 발생할 수 있다는 점이다.

`throws` 랑 `UIImage` 중에 하나는 무조건 리턴 되도록 스위프트가 강제할 것이다.

위 코드에서 조금 특이했던 `await maybeImage?.thumbnail` 은 다음과 같은 `extension` 으로 구현됐다.

<img width="1232" alt="Screen Shot 2022-06-23 at 6 10 18 PM" src="https://user-images.githubusercontent.com/33091784/175308073-64b52d88-def3-4c67-92dd-e6066d13cc1c.png">

- 스위프트 5.5 이후로는 프로퍼티 `getter` 도 `throw` 할 수 있다.
- 
<img width="1232" alt="Screen Shot 2022-06-23 at 6 12 20 PM" src="https://user-images.githubusercontent.com/33091784/175308089-abcfad77-227d-4d8d-a8b1-b4f4548ae002.png">

- 반복문으로도 가능

## `async` 함수가 `suspend` 되다?

<img width="1379" alt="Screen Shot 2022-06-23 at 6 15 10 PM" src="https://user-images.githubusercontent.com/33091784/175308105-b10643d4-2090-4eb5-ab7f-6e7cd5e1bc56.png">

- `thumbnailURLRequest` 가 호출되면 끝날때까지 스레드를 계속 차지한다.
- 끝나면 다시 `fetchThumbnail` 이 스레드를 차지

<img width="1379" alt="Screen Shot 2022-06-23 at 6 21 02 PM" src="https://user-images.githubusercontent.com/33091784/175308126-962cbf38-e7e8-49f3-b7e2-cd4f35acade0.png">

- 위랑 마찬가지로 함수 호출이 끝나면 `fetchThumbnail` 한테 쓰레드를 돌려주지만, `suspend` 를 통해 조금 다르게 동작할 수 있다.
- `fetchThumbnail` 에서 `data(for: request)` 를 호출하면, `suspend` 라는것을 할 수 있다. 이때, 쓰레드를 `fetchThumbnail` 을 돌려주는 것이 아니라, `system` 한테 준다. (이때 `fetchThumbnail` 도 `suspend` 된다)
- `suspend` 라는 것은 `system` 한테 제어를 넘겨줘서 어떤 작업들을 할지 결정할 수 있게 해준다.(스케쥴링?)
- 그래서 `system` 이 `data(for: request)`` 를 다시 `resume` 할 수도 있고, `suspend` 는 이후에 몇번이고 다시 될 수도 있다.
- 무조건 `async` 로 표시되면 `suspend` 되는 것은 아님

```swift
func fetchThumbnail(for id: String) async throws -> UIImage {
	let request = thumbnailURLRequest(for: id)
	let (data, response) = try await URLSesesion.shared.data(for: request) // 1.

	guard (response as? HTTPURLResponse)?.statusCode == 200 else {
		throw FetchError.badID
	}
	let maybeImage = UIIMage(data: data)
	guard let thumbnail = await maybeImage?.thumbnail else {
		throw FetchError.badImage
	}
	return thumbnail
}
```

1. `[URLSession.shared.data](http://URLSession.shared.data)` 가 호출되면, `suspend` (“ `async` 만의 특별한 방법”) 를 통해 해당 쓰레드에서의 작업을 중단한다.
    1. 즉, 쓰레드의 제어권을 시스템한테 넘겨서 해당 메서드(`data()` ) 의 작업을 스케쥴링 시킨다.
    2. 시스템이 판단해서 해당 스레드로 다른 더 중요한 일을 할 수도 있다.
    
    예시 시나리오:
    
    <img width="1379" alt="Screen Shot 2022-06-23 at 7 52 27 PM" src="https://user-images.githubusercontent.com/33091784/175308222-679947e2-1c97-4340-b72b-211c45d94bb7.png">


    
    - `data()` 가 호출된 후에 사용자 입력이 발생
    <img width="1379" alt="Screen Shot 2022-06-23 at 7 51 12 PM" src="https://user-images.githubusercontent.com/33091784/175308187-88e44bea-5183-45d3-a24a-d3c2c02de41e.png">
    
    - 그럼 일단 그게 더 중요하기 때문에 `data` 의 일은 잠시 중단
    - 그다음에 순차적으로 작업들이 완료되면 `fetchThumbnail` 메서드도 반환된다.

# `async/await` 정리

<img width="681" alt="Screen Shot 2022-06-23 at 7 56 18 PM" src="https://user-images.githubusercontent.com/33091784/175308505-90d0d54f-1416-4700-a8e7-4bd3b61c1429.png">

# Testing async code

### Before
<img width="1289" alt="Screen Shot 2022-06-23 at 7 58 02 PM" src="https://user-images.githubusercontent.com/33091784/175308534-ab76cefa-ab24-4533-a727-831c6ed02117.png">

### After

<img width="1289" alt="Screen Shot 2022-06-23 at 7 59 33 PM" src="https://user-images.githubusercontent.com/33091784/175308560-a2b710af-b5ba-42fe-a4f7-f5020f7b0876.png">

# Bridging from sync to async

### Before

<img width="1289" alt="Screen Shot 2022-06-23 at 8 01 05 PM" src="https://user-images.githubusercontent.com/33091784/175308596-733fda94-aa98-471a-9549-4c91938df3c8.png">

# After

<img width="1289" alt="Screen Shot 2022-06-23 at 9 36 50 PM" src="https://user-images.githubusercontent.com/33091784/175308615-35686ed7-d0ce-4089-9be7-1266fd92d99d.png">

이때 `.onAppear` 는 `UI` 관련된거라 메인 스레드..즉, 동기로 작동됨

이런 `sync` 와 `async` 를 연결해주는게 `Task` 함수

### `Task`

클로져 안에 있는 작업을 시스템으로 보내서 다음으로 사용이 가능한 스레드에서 바로 실행한다.

`DispatchQueue.global.async` 와 같은 일을 한다고 보면 된다.

이걸 사용하는 이점은, `sync` 컨텍스트 안에서 `async` 를 사용할 수 있다는 것이다.

<img width="1289" alt="Screen Shot 2022-06-23 at 9 53 52 PM" src="https://user-images.githubusercontent.com/33091784/175308649-226693ab-75eb-4fb5-8e9c-51638985261f.png">

# Async APIs in the SDK

## Before

<img width="1365" alt="Screen Shot 2022-06-23 at 9 56 16 PM" src="https://user-images.githubusercontent.com/33091784/175308681-ad412e17-b07a-4dcd-a50c-c69ea2cc3cab.png">

## After

스위프트 5.5 부터

<img width="1365" alt="Screen Shot 2022-06-23 at 9 56 45 PM" src="https://user-images.githubusercontent.com/33091784/175308703-c03ac882-483b-43af-9e44-3056967c8f10.png">

## Before

<img width="1365" alt="Screen Shot 2022-06-23 at 9 58 11 PM" src="https://user-images.githubusercontent.com/33091784/175308727-0fc354f7-05da-4007-9209-c1c55515957f.png">

## After

### 네이밍 팁

`async` 를 사용하는 함수에서는 앞에 `get` 를 빼라 (왜냐하면 바로 `get` 하지를 못하니까)

![Screen Shot 2022-06-23 at 10 24 57 PM](https://user-images.githubusercontent.com/33091784/175309709-ec092d08-7561-4c9d-96a4-b3654135b5b3.png)

# Async alternatives and continuations

<img width="1365" alt="Screen Shot 2022-06-23 at 10 01 06 PM" src="https://user-images.githubusercontent.com/33091784/175308797-645f02ba-9fb3-49a2-9a80-b6d5cd499380.png">

<img width="1365" alt="Screen Shot 2022-06-23 at 10 02 25 PM" src="https://user-images.githubusercontent.com/33091784/175308826-3f60c0b2-e20f-4171-84e0-233891f96bf8.png">

- `getPresistentPosts` 의 결과를 기다리고있는 `persistentPosts` 한테 반환할 방법을 생각해야된다
- `getPersistentPosts` 가 호출될때 `persistentPosts` 는 현재 `suspended` 돼있다.
- 올바른 데이터를 올바른 타이밍에 `resume` 해야된다

<img width="1365" alt="Screen Shot 2022-06-23 at 10 06 27 PM" src="https://user-images.githubusercontent.com/33091784/175308858-4edf7945-51a3-47e6-b7cc-d06fd2bcaa06.png">

- `await` 다음에 콜이 끝나면 `resume` 이 된다.

<img width="1365" alt="Screen Shot 2022-06-23 at 10 08 49 PM" src="https://user-images.githubusercontent.com/33091784/175308874-cdb41522-6d5a-47d5-9de7-8c9f4094c9ea.png">

<img width="1365" alt="Screen Shot 2022-06-23 at 10 09 06 PM" src="https://user-images.githubusercontent.com/33091784/175308901-6457c62b-1aef-4078-9258-233374415473.png">

<img width="1365" alt="Screen Shot 2022-06-23 at 10 09 32 PM" src="https://user-images.githubusercontent.com/33091784/175310163-e6355972-4cdf-4f1f-8e0f-3265cb15a953.png">

런타임 오류가 발생시켜준다

# Summary

- `Async/await` code is simple easy and safe
- Hundreds of new `async` APIs
- automatic async legacy API imports

# 참고할만한 링크

[https://github.com/apple/swift-evolution/blob/main/proposals/0296-async-await.md](https://github.com/apple/swift-evolution/blob/main/proposals/0296-async-await.md)
