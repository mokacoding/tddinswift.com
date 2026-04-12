---
title: 'Chapter 8 – Exercise Solution'
date: 2021-07-16T01:01:57+00:00
description: "Chapter 8: Testing Code Based on Indirect Inputs – Exercise Solution Chapter 8 introduced the concept of Test Doubles, test-specific equivalents of production dependencies, and learned how to build…"
slug: chapter-8-testing-code-based-on-indirect-inputs
---

Chapter 8 introduced the concept of Test Doubles, test-specific equivalents of production dependencies, and learned how to build a Stub to control the indirect inputs a dependency provides to the System Under Test.

```swift
// MenuFetchingStub.swift
@testable import Albertos
import Combine
import Foundation

class MenuFetchingStub: MenuFetching {

    let result: Result<[MenuItem], Error>

    init(returning result: Result<[MenuItem], Error>) {
        self.result = result
    }

    func fetchMenu() -> AnyPublisher<[MenuItem], Error> {
        return result.publisher
            // Use a delay to simulate the real world async behavior
            .delay(for: 0.1, scheduler: RunLoop.main)
            .eraseToAnyPublisher()
    }
}
```

The exercise in the chapter is a challenge to improve the app’s UX by adding a way for the users to retry the menu fetch call if it fails.

For that, we’ll need a retry button in the view and logic to retry the `MenuFetching` `fetchMenu()` call in the ViewModel.

### Step 0: Action Plan

Before getting coding, let’s draw an action plan for how to perform these changes.\
Building up a list of steps to take is an application of the core principle *Partition Problem and Solve Sequentially*.\
By decomposing a task into steps and writing them down, we free our minds to focus entirely on solving one problem at a time.

We’ll need to:

- Add a method to the ViewModel to retry the call, which the view can call
- To test the behavior, we need to distinguish between multiple `fetchMenu()` calls in the tests
- Make both the initial ViewModel request and the retry one use the same `MenuFetching` `fetchMenu()` logic
- Add a `Button` in the view to retry the call, which will call the ViewModel method

Now that we have our action plan, we can start executing.\
Partitioning the problem of implementing the retry behavior revealed two refactors we’ll need to make.\
For the ViewModel to use the same logic upon the initial fetch and when retrying, that logic needs to be extracted from the `init` method, where it currently lives.\
In the tests, if we want to distinguish between multiple `fetchMenu()` calls, we need first to be able to simulate making multiple calls.\
Our Stub Test Double needs to be able to provide more than one input to the SUT.

We should tackle these refactors before moving on to implementing the retry logic.\
By refactoring in isolation, we can fully leverage the tests for feedback on our code changes.\
Also, by implementing the refactor first, we keep the app in a shippable state.\
When working in a team, you could open a PR with the refactors and have that reviewed in isolation.\
Shipping refactors in dedicated PRs is useful because small PRs are easier to review and get feedback faster.

Suppose we hadn’t taken a minute to write down our action plan. In that case, we might have missed this opportunity for incremental improvements and ended up making those changes while in the middle of making the test pass, adding extra complexity to our work.

### Step 1: Preparatory Refactor — Extract `fetchMenu()` call in ViewModel

Let’s extract the call to `MenuFetching` from the `init` method to a dedicated method so that when we’ll add a retry method to the ViewModel, it’ll be able to call it, too.

```swift
// MenuList.ViewModel.swift
// ...
private let menuFetching: MenuFetching
private let menuGrouping: ([MenuItem]) -> [MenuSection]

init(
    menuFetching: MenuFetching,
    menuGrouping: @escaping ([MenuItem]) -> [MenuSection] = groupMenuByCategory
) {
    self.menuFetching = menuFetching
    self.menuGrouping = menuGrouping

    fetchMenu()
}

private func fetchMenu() {
    menuFetching
        .fetchMenu()
        .map(menuGrouping)
        .sink(
            receiveCompletion: { [weak self] completion in
                guard case .failure(let error) = completion else { return }
                self?.sections = .failure(error)
            },
            receiveValue: { [weak self] value in
                self?.sections = .success(value)
            }
        )
        .store(in: &cancellables)
}
```

Notice how I kept the new `fetchMenu()` `private`.\
In the context of this refactor, we don’t want to add any new internally visible API that could potentially change the app by opening the doors for new behaviors.

To verify this change didn’t affect the app’s behavior, we only need to run the unit tests with the `Cmd U` keyboard shortcut.

### Step 2: Preparatory Refactor – Support multiple values in Stub

To eventually test the retry behavior, we want to send different known values for every `menuFetch()` call to `MenuFetchingStub`.\
This way, we can assert that the retry logic actually started a new menu fetching.

We can simulate that by configuring `MenuFetchingStub` with an array of values to return and make it extract the appropriate one on every `menuFetch()` call.

```swift
// MenuFetchingStub.swift
// ...
class MenuFetchingStub: MenuFetching {

    private(set) var results: [Result<[MenuItem], Error>]

    init(returning result: Result<[MenuItem], Error>) {
        self.results = [result]
    }

    init(returning results: [Result<[MenuItem], Error>]) {
        self.results = results
    }

    func fetchMenu() -> AnyPublisher<[MenuItem], Error> {
        guard let result = results.first else { fatalError() }
        results = Array(results.dropFirst())

        return result.publisher
            // Use a delay to simulate the real world async behavior
            .delay(for: 0.1, scheduler: RunLoop.main)
            .eraseToAnyPublisher()
    }
}
```

The implementation is pretty similar to what we already had, but instead of returning the `Result` stored at `init`, we make the Stub pop the value to return from an array.

Notice that, by leaving the `init` method that takes a single `Result` as input, we don’t need to change the test code using this Stub.

Once again, if the tests pass, our refactor was successful.

### Step 3: Write a test for the retry logic

In the previous steps, we put in place all the infrastructure required to write a test for the retry behavior.\
Let’s get to it, then.

```swift
// MenuList.ViewModelTests.swift
// ...
func testRetryMakesNewFetchRequest() {
    let menuFetchingStub = MenuFetchingStub(
        returning: [.failure(TestError(id: 123)), .failure(TestError(id: 234))]
    )

    let viewModel = MenuList.ViewModel(
        menuFetching: menuFetchingStub,
        menuGrouping: { _ in [] }
    )

    let expectation = XCTestExpectation(description: "Publishes an error")

    let cancellable = viewModel
        .$sections
        .dropFirst()
        .sink { value in
            guard case .failure(let error) = value else {
                return XCTFail("Expected a failing Result, got: \(value)")
            }

            XCTAssertEqual(error as? TestError, TestError(id: 123))
            expectation.fulfill()
        }

    wait(for: [expectation], timeout: 1)

    // Cancel the previous subscription; otherwise, it will be called as well and fail the test
    // because the error we should get on the retry is different from the first one.
    cancellable.cancel()

    let expectation2 = XCTestExpectation(description: "Retries and receives a different value")

    viewModel
        .$sections
        // We `dropFirst()` here too, to discard the value stored in `sections`
        // from the previous fetch.
        .dropFirst()
        .sink { value in
            guard case .failure(let error) = value else {
                return XCTFail("Expected a failing Result, got: \(value)")
            }

            XCTAssertEqual(error as? TestError, TestError(id: 234))
            expectation2.fulfill()
        }
        .store(in: &cancellables)

    viewModel.retry()

    wait(for: [expectation2], timeout: 1)
}

// MenuList.ViewModel.swift
// ...
func retry() {}
```

The test above:

1.  Configures the Stub to return two different errors
2.  Waits for the ViewModel to receive the first error
3.  Defines a new expectation to receive the second error
4.  Exercises the `retry()` method (for which we only have an empty implementation)
5.  Waits for the new expectation

The test fails because of the empty `retry()` implementation.\
That’s what we wanted: first, write the test, then make it pass.\
Seeing the test fail also validates its ability to recognize incorrect behavior in the SUT.

### Step 4: Make the test pass

Thanks to the refactor we did in Step 1, making the test pass require nothing more that calling the ViewModel `fetchMenu()` method from `retry()`:

```swift
func retry() {
    fetchMenu()
}
```

### Step 5: Update the view

We now need to add a button to retry to the view.

```swift
// MenuList.swift
// ...
case .failure(let error):
    VStack {
        Text("An error occurred:")
        Text(error.localizedDescription).italic()
        Button("Retry", action: viewModel.retry)
    }
```

Notice that we don’t need any extra logic to decide when to show the retry button because we are using `Result` to model the API call output.\
The app will show the button only if the ViewModel publishes a `failure` through its `@Published` `sections` property.

Also, notice that because the retry method signature is `retry()`, which is equivalent to `retry() -> Void`, we can pass it directly as the `action` parameter for the `Button`.\
I find this neater than wrapping the call in an unnecessary closure:

```swift
Button("Retry", action: { viewModel.retry() })
// vs
Button("Retry", action: viewModel.retry)
```

### Step 6: Manual test – Optional!

There’s no real reason to test this retry logic we just implemented manually.\
Because the bulk of the logic lives in the ViewModel, we have it covered by the unit tests we wrote to guide us in the development.

It’s true that the view code is not tested and that we could theoretically have called the wrong ViewModel method.\
Realistically, though, it’s more likely that we’ll forget to add the `Button` view than call the wrong method.\
Forgetting steps is another reason I recommend starting with an action plan.\
When we write down each step we want to take, we’re less likely to forget about one.

Still, let’s imagine we want to showcase the retry behavior to Alberto.\
At this point of the app-building journey in the book, we don’t yet have a real API to call – *and even if we had it, one would hope it didn’t constantly fail!*

We can simulate the error behavior by updating `MenuFetchingPlaceholder` to return an error on the first call and the dummy menu on the following calls.

```swift
// MenuFetchingPlaceholder.swift
import Combine
import Foundation

class MenuFetchingPlaceholder: MenuFetching {

    private var fetchCallCount = 0

    func fetchMenu() -> AnyPublisher<[MenuItem], Error> {
        let result: Result<[MenuItem], Error>
        // Return an error on the first fetch
        if fetchCallCount == 0 {
            result = .failure(
                NSError(
                    domain: "test",
                    code: 1,
                    userInfo: [NSLocalizedDescriptionKey: "Test error"]
                )
            )
        } else {
            result = .success(menu)
        }

        fetchCallCount += 1

        return Future { $0(result) }
            // Use a delay to simulate async fetch
            .delay(for: 0.5, scheduler: RunLoop.main)
            .eraseToAnyPublisher()
    }
}
```

<figure class="aligncenter is-resized">
<img src="https://web.archive.org/web/20230201195124im_/https://i0.wp.com/tddinswift.com/wp-content/uploads/2021/07/albertos-retry-demo.gif?resize=209%2C456&amp;ssl=1" class="jetpack-lazy-image" width="209" height="456" alt="Animated GIF showing the retry flow in action" />
<img src="https://web.archive.org/web/20230201195124im_/https://i0.wp.com/tddinswift.com/wp-content/uploads/2021/07/albertos-retry-demo.gif?resize=209%2C456&amp;ssl=1" width="209" height="456" alt="Animated GIF showing the retry flow in action" />
</figure>

## Alternative Test Implementation

Here are two alternative implementations for the test just for fun – *this is bonus content, after all*.

### Alternative 1: Cancel all `cancellables`

In the test for the retry behavior, we stored the `AnyCancellable` returned by the first `sink` call.\
We need that first subscription to verify the retry behavior by asserting that the ViewModel publishes two different values, hence made two calls to `MenuFetching` and received two different responses, but we need to cancel it to avoid it being called when we retry.

A different approach to canceling the subscription is to cancel *all* the subscriptions that the test stored.

```swift
// MenuList.ViewModelTests.swift
// ...
let expectation = XCTestExpectation(description: "Publishes an error")

viewModel
    .$sections
    .dropFirst()
    .sink { value in
        guard case .failure(let error) = value else {
            return XCTFail("Expected a failing Result, got: \(value)")
        }

        XCTAssertEqual(error as? TestError, TestError(id: 123))
        expectation.fulfill()
    }
    .store(in: &cancellables)

wait(for: [expectation], timeout: 1)

cancellables.forEach { $0.cancel() }
```

It’s safe to cancel all of the other `cancellables` because tests run sequentially in the context of the same test class.\
Even when running tests in parallel, Xcode will parallelize different test classes;\
Xcode won’t distribute tests from the same class across different Simulators.\
From the Xcode 10 release notes:

> Test parallelization occurs by distributing the test classes in a target across\
> multiple runner processes.

I somehow preferred storing the `AnyCancellable` value locally when writing the test, but I don’t have a strong rationale for choosing one option instead of the other.

### Alternative 2: Use a single `sink`

A different way to deal with multiple calls to the ViewModel triggering different subscriber closures is to *use a single subscriber*.\
Remember the saying: the best way to solve a problem is not to have it.

```swift
let expectation = XCTestExpectation(description: "Publishes an error")
let expectation2 = XCTestExpectation(description: "Retries and gets a different value")

var receivedErrors: [Error] = []

viewModel
    .$sections
    .dropFirst()
    .sink { value in
        guard case .failure(let error) = value else {
            return XCTFail("Expected a failing Result, got: \(value)")
        }
        guard receivedErrors.count < 2 else {
            return XCTFail("Expected only two errors, got a third")
        }

        receivedErrors.append(error)

        if receivedErrors.count == 1 { expectation.fulfill() }
        if receivedErrors.count == 2 { expectation2.fulfill() }
    }
    .store(in: &cancellables)

wait(for: [expectation], timeout: 1)

viewModel.retry()

wait(for: [expectation2], timeout: 1)

// This check is a bit redundant, given the expectation fulfillment conditions above
XCTAssertEqual(receivedErrors.count, 2)
XCTAssertEqual(receivedErrors[safe: 0] as? TestError, TestError(id: 123))
XCTAssertEqual(receivedErrors[safe: 1] as? TestError, TestError(id: 234))
```

This approach makes the test flow less linear because we need to manage both expectations in the same `sink` even though they refer to sequential calls separated by `retry()`.\
On the other hand, the test is much more compact this way.

All the alternatives produce the same result. Which option to choose is up to your taste; there’s no right or wrong answer.

As an aside, without the kind of unit tests coverage that practicing TDD produces, playing around with implementations this way becomes much more cumbersome, and people will tend not to do it.\
It’s a shame because it’s only by experimenting and trying different approaches that we can learn and grow.
