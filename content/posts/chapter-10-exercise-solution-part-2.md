---
title: 'Chapter 10 – Exercise Solution Part 2'
date: 2021-07-21T20:38:19+00:00
description: "Chapter 10 – Testing Network Code – Exercises Solution Part 2: How to test the URLRequest configuration with a Spy Test-Driven Development in Swift’s Chapter 10 shows how to use TDD to implem…"
slug: chapter-10-exercise-solution-part-2
---

Part 2: How to test the `URLRequest` configuration with a Spy

[*Test-Driven Development in Swift*](/)‘s Chapter 10 shows how to use TDD to implement the kernel of an application’s networking logic. The resulting implementation is necessarily simple; because of the book’s space constraints, it only services one API. At the end of the chapter, I suggested two exercises to improve the implementation: add a configurable base URL and another API call.

[Previously](/chapter-10-exercise-solution-part-1/), we saw how to refactor `MenuFetcher` adding a base URL parameter to its `init` using TDD and the Stub we wrote in Chapter 10.\
In this bonus content post, we’ll see a different approach, using a Spy Test Double. You can learn all about Spies in Chapter 12.

## Step 0: Action plan

Here’s a reminder of the action plan we devised in the [previous post](/chapter-10-exercise-solution-part-1/).

To add a base URL property to `MenuFetcher`, we’ll need to update its initialization code. We’ll then need to replace the hardcoded URL used to build the `URLRequest` with one created by interpolating the base URL value with the endpoint path.

To test this new behavior, we can build a Spy for the underlying `NetworkFetching` service and use it to record the `URLRequest` that `MenuFetcher` creates.

## Base URL with Spy – Step 1: Create the Spy Test Double

Quick reminder: a Spy is a Test Double, a “test-specific equivalent” of a dependency of the System Under Test, aimed at allowing us to verify the *indirect outputs* the SUT produces.

Indirect outputs are anything that is not a return value of a method call,\
such as a change in global state or calling a method on another component.

In our case, the indirect output is the `URLRequest` that `MenuFetcher` builds and passes to `NetworkFetching` for execution.

```swift
// NetworkFetchingSpy.swift
@testable import Albertos
import Combine
import Foundation

class NetworkFetchingSpy: NetworkFetching {

    private(set) var request: URLRequest?

    func load(_ request: URLRequest) -> AnyPublisher<Data, URLError> {
        self.request = request

        return Result<Data, URLError>
            .failure(URLError(.badURL))
            .publisher
            // Use a delay to simulate the real world async behavior
            .delay(for: 0.01, scheduler: RunLoop.main)
            .eraseToAnyPublisher()
    }
}
```

The Spy implementation is similar to the Stub from Chapter 10. Like the Stub, it, too, simulates the real network request by leveraging `Result`‘s capability to generate a `Publisher` and add a delay to introduce an async component in the code execution.

In the `load(_:)` implementation, the Spy stores the `URLRequest` input parameter in a `private(set)` variable so that consumers can read it but not accidentally modify it.

Unlike the Stub, though, the Spy doesn’t allow configuring the `load(_:)` return value but has a hardcoded return value. In this implementation I chose to return an error, but you could as well return a successful value by making `MenuFetcher` conform to `Encodable` and using:

```swift
.success(try! JSONEncoder().encode([MenuItem.fixture()]))
```

It doesn’t matter what the Spy returns because the test won’t be looking at it; they’ll only use the probes the Spy offers to measure the SUT side-effects.

## Base URL with Spy – Step 2: Write a test for the `URLRequest` composition

Like in the [Stub approach](/chapter-10-exercise-solution-part-1/), let’s write a test with the code we have already and use it as a guide to make sure adding the base URL parameter to `MenuFetcher` doesn’t break its behavior.

Here’s how to use the Spy to check the `URLRequest` that `MenuFetcher` produces:

```swift
// MenuFetcherTests.swift
// ...
func testUsesGivenBaseURLInRequest() {
    let spy = NetworkFetchingSpy()
    let menuFetcher = MenuFetcher(networkFetching: spy)

    let expectation = XCTestExpectation(description: "Request succeeds")

    menuFetcher.fetchMenu()
        .sink(
            receiveCompletion: { _ in expectation.fulfill() },
            receiveValue: { _ in }
        )
        .store(in: &cancellables)

    wait(for: [expectation], timeout: 1)

    XCTAssert(spy.request?.url?.absoluteString == "https://raw.githubusercontent.com/mokagio/tddinswift_fake_api/trunk/menu_response.json")
}
```

What the test does is arranging `MenuFetcher` with `NetworkFetchingSpy`, acting on it via `fetchMenu()`, and asserting how it formed the `URLRequest` after waiting for the fetch to complete.

As I mentioned in the book, while using vanilla XCTest keeps your test suite setup lean, this testing framework from Apple has poor ergonomics when it comes to its assertion APIs. Using Nimble, discussed in the book’s Appendix B, makes the test clearer:

```swift
// MenuFetcherTests.swift
// ...
import Nimble

// ...
func testUsesGivenBaseURLInRequest() {
    let spy = NetworkFetchingSpy()
    let menuFetcher = MenuFetcher(networkFetching: spy)

    _ = menuFetcher.fetchMenu()

    expect(spy.request?.url?.absoluteString)
        .toEventually(equal("https://raw.githubusercontent.com/mokagio/tddinswift_fake_api/trunk/menu_response.json"))
}
```

## Base URL with Spy — Step 3: Add the base URL parameter to `MenuFetcher`

Now that we have a test that verifies the behavior we aim to modify, we can set out to work with the peace of mind of quickly being able to verify whether our changes are correct.

The refactoring steps are the same as what we did in the [previous article](/chapter-10-exercise-solution-part-1/), eventually resulting in the following production and test code:

```swift
// MenuFetcher.swift
import Combine
import Foundation

class MenuFetcher: MenuFetching {

    let networkFetching: NetworkFetching
    let baseURL: URL

    init(
        baseURL: URL = URL(string: "https://raw.githubusercontent.com/mokagio/tddinswift_fake_api/trunk")!,
        networkFetching: NetworkFetching = URLSession.shared
    ) {
        self.baseURL = baseURL
        self.networkFetching = networkFetching
    }

    func fetchMenu() -> AnyPublisher<[MenuItem], Error> {
        let request = URLRequest(url: baseURL.appendingPathComponent("menu_response.json"))

        return networkFetching.load(request)
            .decode(type: [MenuItem].self, decoder: JSONDecoder())
            .eraseToAnyPublisher()
    }
}
```

```swift
// MenuFetcherTests.swift
// ...
func testUsesGivenBaseURLInRequest() throws {
    let spy = NetworkFetchingSpy()
    let menuFetcher = MenuFetcher(
        baseURL: try XCTUnwrap(URL(string: "https://test.fake")),
        networkFetching: spy
    )

    _ = menuFetcher.fetchMenu()

    expect(spy.request?.url?.absoluteString)
        .toEventually(equal("https://test.fake/menu_response.json"))
}
```

## Spying vs. Stubbing

So, which approach is better to help us verify the `URLRequest` creation?\
Stub or Spy?

As a reminder, here’s the test implemented using a Stub Test Double from the [previous post](/chapter-10-exercise-solution-part-1/), converted to use Nimble, too, to make the comparison fair:

```swift
// MenuFetcherTests.swift
// ...
func testUsesGivenBaseURLInRequest() throws {
    let baseURL = try XCTUnwrap(URL(string: "https://test.url"))
    let url = baseURL.appendingPathComponent("menu_response.json")
    let json = """
[
    { "name": "a name", "category": "a category", "spicy": true, "price": 1.0 }
]
"""
    let data = try XCTUnwrap(json.data(using: .utf8))
    let networkFetchingStub = NetworkFetchingStub(
        returning: .success(data),
        for: URLRequest(url: url)
    )
    let menuFetcher = MenuFetcher(baseURL: baseURL, networkFetching: networkFetchingStub)

    waitUntil { done in
        menuFetcher.fetchMenu()
            .sink(
                receiveCompletion: { _ in },
                receiveValue: { _ in
                    // We don't care about the values received. We're only interested to know
                    // that we're here instead than in the failure branch, which means the
                    // request received by the Stub matched our expectation.
                    done()
                }
            )
            .store(in: &self.cancellables)
    }
}
```

The contrast between approaches is striking, and so is the winner.\
The Spy.

Quoting from Gerard Meszaros’ *xUnit Test Patterns*, one of the classics in testing literature, a Spy offers ” an observation point that exposes the indirect outputs of the SUT so they can be verified.” That’s precisely what we need to test a behavior like calling an internal object with a specific input value.

The difference in the simplicity and readability of the two tests is why I follow Meszaros Test Doubles naming convention, rather than calling them all “Stub” or “Mock,” like I’ve seen in many codebases. Different behaviors require different testing strategies. Stubs, Spies, Fakes, Doubles, each are sharp tools well suited for different circumstances. Choosing the proper Test Double for the job will make your tests simple and communicate clear intent on the testing strategy.
