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
import Foundation

class MenuFetchingStub: MenuFetching {

    let result: Result<[MenuItem], Error>

    init(returning result: Result<[MenuItem], Error>) {
        self.result = result
    }

    func fetchMenu() async throws -> [MenuItem] {
        try result.get()
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

    Task { await fetchMenu() }
}

private func fetchMenu() async {
    do {
        let items = try await menuFetching.fetchMenu()
        sections = .success(menuGrouping(items))
    } catch {
        sections = .failure(error)
    }
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

    func fetchMenu() async throws -> [MenuItem] {
        guard let result = results.first else { fatalError() }
        results = Array(results.dropFirst())

        return try result.get()
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
<!-- VERIFY: prefer Task.sleep or confirmation in async tests -->
@Test func retryMakesNewFetchRequest() async throws {
    let menuFetchingStub = MenuFetchingStub(
        returning: [.failure(TestError(id: 123)), .failure(TestError(id: 234))]
    )
    let viewModel = MenuList.ViewModel(
        menuFetching: menuFetchingStub,
        menuGrouping: { _ in [] }
    )

    // First fetch (started in init)
    try await Task.sleep(for: .milliseconds(150)) // allow the fetch to complete
    guard case .failure(let firstError) = viewModel.sections else {
        Issue.record("Expected a failing state, got \(viewModel.sections)")
        return
    }
    #expect(firstError as? TestError == TestError(id: 123))

    // Retry
    viewModel.retry()
    try await Task.sleep(for: .milliseconds(150))
    guard case .failure(let secondError) = viewModel.sections else {
        Issue.record("Expected a failing state after retry, got \(viewModel.sections)")
        return
    }
    #expect(secondError as? TestError == TestError(id: 234))
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

Thanks to the refactor we did in Step 1, making the test pass requires nothing more than calling the ViewModel `fetchMenu()` method from `retry()`:

```swift
func retry() {
    Task { await fetchMenu() }
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
import Foundation

class MenuFetchingPlaceholder: MenuFetching {

    private var fetchCallCount = 0

    func fetchMenu() async throws -> [MenuItem] {
        defer { fetchCallCount += 1 }

        // Simulate async fetch
        try await Task.sleep(for: .milliseconds(500))

        // Return an error on the first fetch
        if fetchCallCount == 0 {
            throw NSError(
                domain: "test",
                code: 1,
                userInfo: [NSLocalizedDescriptionKey: "Test error"]
            )
        }
        return menu
    }
}
```

<!-- TODO(TODO.md): restore `albertos-retry-demo.gif` — asset wasn't captured on wayback.
<figure class="aligncenter is-resized">
<img src="https://web.archive.org/web/20230201195124im_/https://i0.wp.com/tddinswift.com/wp-content/uploads/2021/07/albertos-retry-demo.gif?resize=209%2C456&amp;ssl=1" class="jetpack-lazy-image" width="209" height="456" alt="Animated GIF showing the retry flow in action" />
<img src="https://web.archive.org/web/20230201195124im_/https://i0.wp.com/tddinswift.com/wp-content/uploads/2021/07/albertos-retry-demo.gif?resize=209%2C456&amp;ssl=1" width="209" height="456" alt="Animated GIF showing the retry flow in action" />
</figure>
-->


## Alternative Test Implementation

Here are two alternative implementations for the test just for fun – *this is bonus content, after all*.

### Alternative: Inspect via `confirmation`

With Combine-based tests, this section used to discuss canceling old subscriptions or sharing a single `sink` across both expected emissions. Under `async`/`await`, those concerns largely disappear — we just read the state after each fetch completes.

If the ViewModel still exposes a `$sections` publisher (because Ch 7 in v2 keeps `@Published`/`ObservableObject` per the TOC), we can lean on Swift Testing's `confirmation` to wait for the expected number of value emissions instead of sleeping:

```swift
@Test func retryEmitsTwoFailures() async throws {
    let menuFetchingStub = MenuFetchingStub(
        returning: [.failure(TestError(id: 123)), .failure(TestError(id: 234))]
    )
    let viewModel = MenuList.ViewModel(
        menuFetching: menuFetchingStub,
        menuGrouping: { _ in [] }
    )

    await confirmation("Emits two failures", expectedCount: 2) { confirm in
        var cancellables: Set<AnyCancellable> = []
        viewModel.$sections
            .dropFirst()
            .sink { value in
                if case .failure = value { confirm() }
            }
            .store(in: &cancellables)

        try? await Task.sleep(for: .milliseconds(200))
        viewModel.retry()
        try? await Task.sleep(for: .milliseconds(200))
    }
}
```

The `confirmation` approach makes the test's intent clearer: we expect two failure emissions.

All the alternatives produce the same result. Which option to choose is up to your taste; there’s no right or wrong answer.

As an aside, without the kind of unit tests coverage that practicing TDD produces, playing around with implementations this way becomes much more cumbersome, and people will tend not to do it.\
It’s a shame because it’s only by experimenting and trying different approaches that we can learn and grow.
