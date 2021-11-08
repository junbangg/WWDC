# Engineering for Testability

## Testable app code

### Structure of a Unit Test
```swift
func testArraySorting() {
    // Arrange
    let input = [1, 7, 6, 3, 10]
    // Act
    let output = input.sorted()
    // Assert
    XCTAssertEqual(output, [1, 3, 6, 7, 10])
}
```
Arrange-Act-Assert 패턴
- Prepare Input
- Run the code being tested
- Verify Output

(Given-When-Then 이라고도 함)

### Characteristics of Testable Code
- Control over inputs
- visibility into outputs
- No hidden state
  - Avoids relying on internal state that can affect the code's behaviour later on
  
### Testability Techniques

- Protocols and parameterization
  - Reduce references to shared instances
  - accept parametrized input
    - also known as a **dependency injection**
  - Introduce a protocol
  - Create a testing implementation
    - 필요한 기능을 `Protocol` 로 만들어놓고, 해당 프로토콜을 채택하는 `MockClass` 를 사용해서 테스트 수행
  
- Separating logic and effects
  - 테스트가 다른 코드에 의존하지 않도록 해야한다
    - 테스트의 속도를 저하시킬 수 있는 요소들은 최대한 분리
  - Extract algorithms into separate types
    - functional style with value types
    - allows for straightforward unit tests that exercise the algorithm in as much detail as required
  - Thin layer on top to execute effects

## Scalable test code

### Balance UI and unit tests
  - Unit tests are great for testing small, hard-to-reach code paths
  - UI tests are better at testing integration of larger pieces
### Code to help UI tests scale
  - Abstracting UI element queries
    - store parts of queries in a variable
    - wrap complex queries in utility methods
    - reduce noise and clutter in UI tests
  - Creating objects and utility functions
    - Encapsulate common testing workflows
    - Cross-platform code sharing
    - Improves maintainability
    - 가독성을 높인다(Scalability와 긴밀히 연관됨)
    - 나중에 바꿀게 있으면 편하다
    - `XCTContent.runActivity` 로 loggin을 Nesting 시킬 수 있다.(WWDC17 What's new in testing)
  - Utilizing shortcuts
    - Avoid working through menu bar for macOS
    - make code more compact
      - increases readability

### Importance of Quality of test code
- Important to consider even though it isn't shipping
- Test code should support the evolution of your app
- **Coding principles in app code also apply to test code**
- **Tests should not be an afterthought**