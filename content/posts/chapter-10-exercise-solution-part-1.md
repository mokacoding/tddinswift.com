---
title: 'Chapter 10 – Exercise Solution Part 1'
date: 2021-07-20T00:43:03+00:00
description: "Chapter 10 – Testing Network Code – Exercises Solution Part 1: How to test the URLRequest configuration with a Stub It’s rare to find an app that doesn’t rely on a backend service to re…"
slug: chapter-10-exercise-solution-part-1
edition_compatibility: ["v2"]
---

## Part 1: How to test the `URLRequest` configuration with a Stub

It’s rare to find an app that doesn’t rely on a backend service to read or store information these days. Networking is one part of the application development where good design and thorough testing are crucial because of its foundational importance for the app’s functionalities and the inherent fallibility of the network infrastructure outside the reach of the application code.

Chapter 10 showed how to apply the concepts of Dependency Inversion, Dependency Injection, and Test Doubles to the design of the networking part of Alberto’s app. The chapter closes with two fun exercises to make the necessarily constrained example code more versatile: adding an API endpoint to fetch data from and updating `MenuFetcher` to be configured with a base URL.

Let’s start with adding a base URL because it gives me a way to show you how to perform a *Test-Driven Refactor*.

## Base URL – Step 0: Make an action plan

Adding a base URL property to `MenuFetcher` means we’ll need to update its initialization code. We’ll then need to replace the hardcoded URL used to build the `URLRequest` with one created by interpolating the base URL value with the endpoint path.

How can we write a test to verify `MenuFetcher` uses the given base URL and appends the correct endpoint? We’ll need a way to inspect the `URLRequest` `MenuFetcher` generates. I can think of two ways to do that.\
One is to follow the hint from the book:

> Modify the Stub to expect a `URLRequest` and `Result` pair and only return the given `Result` when the `URLRequest` passed to `load(_:)` matches the given one.

The other option is to use a Spy Test Double, which we discuss in Chapter 12. In this post, we’ll work with the Stub, in the next, with the Spy.

## Base URL with Stub – Step 1: Update the Stub

We want our `NetworkFetchingStub` to return a given `Result` for a given `URLRequest`. That means we need to:

- Store the `URLRequest` as an `init` parameter together with the `Result`.
- Compare the request used to call the `load(_:)` method with the stored one, and only return the `Result` if they match.

What’s a small step we can take? We can start by adding the `URLRequest` parameter to the Stub in a way that doesn’t require updating the code that uses it:

```swift
// NetworkFetchingStub.swift
// ...
class NetworkFetchingStub: NetworkFetching {

    private let result: Result<Data, URLError>
    private let request: URLRequest?

    init(returning result: Result<Data, URLError>, for request: URLRequest? = .none) {
        self.request = request
        self.result = result
    }

    // ... implementation below ...
}
```

If you run the tests, you’ll see they still compile and pass because, even though we added a new `init` parameter to `NetworkFetchingStub`, the default value means none of the existing code had to change.

We can then move on to adding the logic to check the request sent to the `load(_:)` method:

```swift
// NetworkFetchingStub.swift
// ...
func load(_ request: URLRequest) async throws -> Data {
    switch self.request {
    case .none:
        return try result.get()
    case .some(let storedRequest) where storedRequest == request:
        return try result.get()
    default:
        throw URLError(.unknown)
    }
}
```

Once again, the tests still pass because of the default value for the stored `URLRequest`. It’s worth noting that this implementation makes the assumption we’ll never need to test against code receiving `URLError` `.unknown`. This assumption is dangerous because it’s encoded in the Stub’s implementation. A user of this Test Double would have to look at its source code to learn about it. I think this is a reasonable tradeoff at this point, considering the few usages of this Stub and the fact that our test suite is small. If we were working with a team of developers or a broader test suite, this assumption would be dangerous: sooner or later, someone would write code that doesn’t work because they didn’t know about it.

## Base URL with Stub – Step 2: Write a test for the `URLRequest` composition

Armed with our refined Stub, we can write a test to verify the URL in the request `MenuFetcher` sends to `NetworkFetching`.

```swift
// MenuFetcherTests.swift
// ...
@Test func usesGivenBaseURLInRequest() async throws {
    let url = try #require(URL(string: "https://s3.amazonaws.com/mokacoding/menu_response.json"))
    let json = """
    [
        { "name": "a name", "category": "a category", "spicy": true, "price": 1.0 }
    ]
    """
    let data = try #require(json.data(using: .utf8))
    let networkFetchingStub = NetworkFetchingStub(
        returning: .success(data),
        for: URLRequest(url: url)
    )
    let menuFetcher = MenuFetcher(networkFetching: networkFetchingStub)

    // We don't care about the values received. We're only interested to know
    // the call completes without throwing, which means the request received by
    // the Stub matched our expectation.
    _ = try await menuFetcher.fetchMenu()
}
```

Run the test, and you’ll see it passes. That’s okay because we’re testing logic that exists already. Still, as I mentioned early in the book, *you should never trust a test no one has seen failing*. If you’re paranoid like me, you might want to tweak the expected URL and verify the test fails before moving forward.

## Base URL with Stub – Step 3: Add the base URL parameter to `MenuFetcher`

We now have a test that verifies how `MenuFetcher` builds the `URLRequest` it makes: we can refactor the implementation with confidence.

I like to move in small steps, each backed by a test run, to know I didn’t break the app’s behavior. Before adding a new input parameter to `MenuFetcher`, let’s implement the `URLRequest` `URL` interpolation logic.

We can start by defining the `baseURL` locally within `fetchMenu()`:

```swift
// MenuFetcher.swift
// ...
func fetchMenu() async throws -> [MenuItem] {
    let baseURL = URL(string: "https://raw.githubusercontent.com/mokagio/tddinswift_fake_api/trunk")!
    let request = URLRequest(url: baseURL.appendingPathComponent("menu_response.json"))

    let data = try await networkFetching.load(request)
    return try JSONDecoder().decode([MenuItem].self, from: data)
}
```

Then we can extract the value as an instance constant:

```swift
// MenuFetcher.swift
// ...
class MenuFetcher: MenuFetching {

    let networkFetching: NetworkFetching
    let baseURL = URL(string: "https://raw.githubusercontent.com/mokagio/tddinswift_fake_api/trunk")!

    // ...

    func fetchMenu() async throws -> [MenuItem] {
        let request = URLRequest(url: baseURL.appendingPathComponent("menu_response.json"))

        let data = try await networkFetching.load(request)
        return try JSONDecoder().decode([MenuItem].self, from: data)
    }
}
```

Then we can move the value definition into the `init` method:

```swift
// MenuFetcher.swift
// ...
let networkFetching: NetworkFetching
let baseURL: URL

init(
    baseURL: URL = URL(string: "https://raw.githubusercontent.com/mokagio/tddinswift_fake_api/trunk")!,
    networkFetching: NetworkFetching = URLSession.shared
) {
    self.baseURL = baseURL
    self.networkFetching = networkFetching
}
```

All these changes were refactoring in the true sense of the word.\
We updated the code’s implementation without affecting its behavior.

## Base URL with Stub – Step 4: Explicitly assert the behavior in the test

In its current form, the test is implicitly verifying the `URLRequest` creation behavior. The behavior we want to test is that `MenuFetcher` uses the URL given to it during `init`. The test verifies that implicitly because it so happens that the URL it uses is the same as the default value set in the `MenuFetcher` initializer.

We can make the test explicit by using a different URL value:

```swift
// MenuFetcherTests.swift
// ...
@Test func usesGivenBaseURLInRequest() async throws {
    let baseURL = try #require(URL(string: "https://test.url"))
    let url = baseURL.appendingPathComponent("menu_response.json")
    let json = """
    [
        { "name": "a name", "category": "a category", "spicy": true, "price": 1.0 }
    ]
    """
    let data = try #require(json.data(using: .utf8))
    let networkFetchingStub = NetworkFetchingStub(
        returning: .success(data),
        for: URLRequest(url: url)
    )
    let menuFetcher = MenuFetcher(baseURL: baseURL, networkFetching: networkFetchingStub)
    // ...
}
```

This change has the nice side effect of clearly surfacing the endpoint used for the request. The test passes, and our refactor is officially finished.\
We now support injecting a different base URL into `MenuFetcher`, for example, depending on an environment variable to run the application against a staging service.

There’s something I don’t like about how this test turned out, namely that the behavior verification happens as a byproduct of how the Stub works.\
There is no straightforward assertion to verify that the URL used by `MenuFetcher` is the expected one. Instead, we rely on the fact that if the Stub returns a value, the URL must have been the expected one.

If I were to code review a pull request with this change, I would approve it but ask for a follow-up to make the test clearer by using a Spy instead – something I couldn’t suggest in Chapter 10 because, at that point of the book, we hadn’t learned about Spies yet.

How to write the test using a Spy will be the topic of the following bonus content issue. *Stay tuned.*
