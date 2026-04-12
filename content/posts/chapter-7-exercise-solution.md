---
title: 'Chapter 7 – Exercise Solution'
date: 2021-07-11T21:07:10+00:00
description: "Chapter 7: Testing Dynamic Swift Views – Exercise Solution In Chapter 7, we learned how SwiftUI and Combine work together to keep the view up to date seamlessly.We saw how to configure an Observabl…"
slug: chapter-7-exercise-solution
---

In Chapter 7, we learned how SwiftUI and Combine work together to keep the view up to date seamlessly.\
We saw how to configure an `ObservableObject` to expose a `@Published` property and make the view update every time the property changes using `@ObservedObject`.

The synergy between SwiftUI and Combine takes care of all the heavy lifting involved in syncing dynamic views and streamlines how we approach developing the view layer driven by tests.\
In SwiftUI, views are a function of state, not a sequence of events, and we can be confident that every time we feed the same state to a SwiftUI view, it will render it in the same way.\
This simplifies testing because we only need to test the logic that *produces* the state we input in the view.\
We don’t need to worry about all the view layout configuration and state management; the framework does that for us.

In the book, we tested how `MenuItem.ViewModel` publishes its section with the implicit assumption that there will only be one published value.\
That’s the behavior we expect from a network request to an API, an HTTP call that can either succeed or fail.

A Combine `Publisher`, on the other hand, can emit *many* values over time.

## How to test a Combine `Publisher` that emits multiple values

To test a `Publisher` behavior over multiple events, we need a way to store each value it emits.\
Let’s see how to do this by rewriting the test for how `MenuItem.ViewModel` publishes menu sections when the fetch request succeeds.

A `@Published` property requires a default value and will emit that as soon as a subscriber attaches to it.\
In the book, we used `dropFirst` to discard that value and check the first and only value our codes emits through the `@Published` property.

Let’s remove `dropFirst()` and see how our test behaves.

```swift
func testPublishedSections() {
    var receivedMenu: [MenuItem]?
    let expectedSections = [MenuSection.fixture()]

    let spyClosure: ([MenuItem]) -> [MenuSection] = { items in receivedMenu = items
        return expectedSections
    }

    let viewModel = MenuList.ViewModel(
        menuFetching: MenuFetchingPlaceholder(),
        menuGrouping: spyClosure
    )

    let expectation = XCTestExpectation(
        description: "Publishes sections built from received menu and given grouping closure"
    )

    viewModel
        .$sections
        // the .dropFirst() call was here
        .sink { value in
            // Ensure the grouping closure is called with the received menu
            XCTAssertEqual(receivedMenu, menu)
              // ❌ XCTAssertEqual failed: ("nil") is not equal to...

            // Ensure the published value is the result of the grouping closure
            XCTAssertEqual(value, expectedSections)
              // ❌ XCTAssertEqual failed: ("[]") is not equal to...

            expectation.fulfill()
        }
        .store(in: &cancellables)

    wait(for: [expectation], timeout: 1)
}
```

The test fails.\
That’s not surprising because by removing `.dropFirst`, the first value received in the `sink` closure is the default value defined in the ViewModel:

```swift
// MenuItem.ViewModel.swift
import Combine

extension MenuList {

    class ViewModel: ObservableObject {

        @Published private(set) var sections: [MenuSection] = []
```

How can we store all the values emitted by the `@Published` `sections` underlying `Publisher`?\
And how can we verify they match our expectations?

We can collect all the values emitted in an array and inspect them once a certain condition is met.\
By looking at the stored values count, we can determine whether to inspect the array or wait for more values.\
In our case, we expect to receive an empty value first and a full value second.

```swift
func testPublishedSections() {
    var receivedMenu: [MenuItem]?
    let expectedSections = [MenuSection.fixture()]

    let spyClosure: ([MenuItem]) -> [MenuSection] = { items in receivedMenu = items
        return expectedSections
    }

    let viewModel = MenuList.ViewModel(
        menuFetching: MenuFetchingPlaceholder(),
        menuGrouping: spyClosure
    )

    let expectation = XCTestExpectation(
        description: "Publishes sections built from received menu and given grouping closure"
    )

    // This is were we'll collect all the values published by `$sections`
    var values: [[MenuSection]] = []

    viewModel
        .$sections
        .sink { value in
            // 1. Store the new value the Publisher emitted
            values = values + [value]

            // 2. Inspect the array of received values to decide whether to
            // continue collecting value.
            //
            // We expect to receive two values: the first is the empty array default, the
            // second the result of the menu fetch.
            guard values.count == 2 else { return }

            // 3. At this point, the condition on the received values has been
            // met and we can assert the values match our expectations.

            // 4.a. We expect the first value to be an empty array.
            //
            // We cannot use XCTUnwrap here because it throws but `sink` is not declared with
            // `rethrows`.
            //
            // Notice that we could also call force unwrap with `values.first!` because the
            // `guard` above guarantees we'll have exactly two items at runtime.
            //
            // I prefer this more verbose approach because it prevents the tests from crashing
            // in case the `guard` is accidentally removed.
            guard let defaultSections = values.first else {
                return XCTFail("Values has no elements, but expected one")
            }
            XCTAssertTrue(defaultSections.isEmpty)

            // 4.b. We expect the second value to have been constructed using
            // the received sections and the given grouping closure.
            guard let sections = values[safe: 1] else {
                return XCTFail("Expected a value at index 1, got none")
            }

            // Ensure the grouping closure is called with the received menu
            XCTAssertEqual(receivedMenu, menu)
            // Ensure the published value is the result of the grouping closure
            XCTAssertEqual(sections, expectedSections)
            expectation.fulfill()
        }
        .store(in: &cancellables)

    wait(for: [expectation], timeout: 1)
}
```

This new version of the test passes.

To recap: To test a `Publisher` behavior over multiple emitted values, you need to collect the values by subscribing to it with `sink`.\
Once you’ve collected as many values as expected for the behavior under test, you can inspect each value in order and run the appropriate assertions on it.

*For more examples of how to test different Combine `Publisher` behaviors, checkout my Unit Testing Combine Publisher Cheatsheet.*
