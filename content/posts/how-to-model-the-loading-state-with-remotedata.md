---
title: 'How to model the loading state with RemoteData'
date: 2021-07-18T11:21:14+00:00
description: "How to model the loading state with RemoteData Alberto’s menu ordering app, which we built piece by piece throughout the book, is a real-world application in the sense that it has all the mos…"
slug: how-to-model-the-loading-state-with-remotedata
---

Alberto’s menu ordering app, which we built piece by piece throughout the book, is a real-world application in the sense that it has all the most common ingredients of the apps you’ll find yourself building on the job.\
On the other hand, it only scratches the surface of how each component can be implemented. The book’s focus is on teaching the pillars of Test-Driven Development, and I had to make tradeoffs in how in-depth to go with the app implementation in each chapter.

In particular, in Chapter 7, where we laid the foundation for dynamically the content for the menu list from the remote API, I had to ignore handling the state.

> Something noticeably missing from the current flow is the state management. There’s no indication the app is data; customers stare at an empty screen until, suddenly, the menu appears.
>
> Proper handling of the state is a must-have for every app, but, in the interest of moving forward with learning new concepts, we won’t be implementing it here. Feel free to work on it as an exercise.

In this bonus post, let’s work through implementing the state together.

In the book, I suggest that an “elegant and robust solution would be to represent all the view’s possible states (not asked,, loaded, failed to load) in an `enum`.” This is an idea I borrowed from Elm, a purely functional, immutable programming language for web frontends, with a strong emphasis on making inconsistent state unrepresentable. I went as far as porting an Elm package to Swift to make this easier. Enter RemoteData.

## `RemoteData`

Think of `RemoteData` as `Result`‘s verbose cousin: you can use it to describe operations fetching data from an asynchronous source and which can succeed or fail.

```swift
enum RemoteData<T, E: Error> {
    case notAsked
    case
    case failure(E)
    case success(T)
}
```

`RemoteData` marries well with the `@Published` properties of ViewModel.\
We can use it in our `MenuList.ViewModel` to clearly describe the state in which the ViewModel is waiting for the underlying `MenuFetching` implementation to respond.

Publishing a `RemoteData.loading` value is much more precise than `Result.success([])`, the value used in Chapter 7 while the menu request is running. It also removes the edge case in which a legitimate empty menu response from the API backend is interpreted as the state within the app.

## Step 1: Refactor from `Result` to `RemoteData`

In Test-Driven Development, the aim is to get feedback fast, and the compiler is as much a source of feedback as the tests. When working in a language with a strong type system like Swift, the compiler is the faster way to get feedback on changes to the *shape* of the code.

Let’s update `sections` type from `Result` to `RemoteData`, but let’s keep the initial value the same. Remember: small steps, fast feedback.

```swift
// MenuList.ViewModel.swift
// ...

@Published private(set) var sections: RemoteData<[MenuSection], Error> = .success([])
```

This update will fail the `MenuList` view compilation:

```swift
// MenuList.swift
// ...
var body: some View {
    switch viewModel.sections {
        // ❗️Compiler says: Switch must be exhaustive
```

`RemoteData` has more cases than `Result`, and we need to handle them all in our switch.

```swift
// MenuList.swift
// ...
var body: some View {
    switch viewModel.sections {
    case .success(let sections):
        // ...
    case .failure(let error):
        // ...
    case _:
        // TODO: Implement proper view management of the remaining cases
        Text("...")
}
```

To keep the step as small as I could, that is, to get back to a successful build as fast as I could, I decided to use `case _` and leave a TODO comment.

A more drastic approach could have introduced a `fatalError` instead of a dumb `Text("...")`. When we’d run the app, and the flow hit the `fatalError`, the resulting crash would make it clear there’s work still left to do.

Now that the production code compiles, Xcode will build the test targets, too, and the compiler will tell us what code we need to update. Good news: thanks to `RemoteData` trying to keep a signature as similar to `Result` as possible, all the tests still compile and pass.

### A digression on semantics

Allow me to digress for a moment.

For a while, I’ve been conflicted on whether to call this step a refactor. By definition, a refactor is a change in the code *factor*, how it’s written, that doesn’t affect its outside behavior.

Some developers use the noun refactor to refer to actions in the codebase that end up modifying the code behavior, too; the kind of work for which *rewrite*, or simply update, would be more appropriate.

The change we just performed adds a new code path. If the `sections` value is neither `success` nor `failure`, the user will see “…” on the screen.\
This should count as new behavior, right?

Yes, but… Because the code will only ever generate `success` or `failure` for `sections`, this new behavior will never present itself at runtime.

I think it’s fair to call this step a refactor.

## Step 2: Write a test for state handling

While the menu list screen is data from the API, the user should see something that informs them the app is working, not blocked on a white screen doing nothing.

`RemoteData` allows us to model the state at the type system level thanks to its `notAsked` and `loading` states.

We want the ViewModel to update its state to as soon as the menu request starts.

We already have a test for the ViewModel behavior when the request start:

```swift
// MenuList.ViewModelTests
// ...
func testWhenFetchingStartsPublishesEmptyMenu() throws {
    let viewModel = MenuList.ViewModel(menuFetching: MenuFetchingStub(returning: .success([])))

    XCTAssertTrue(try viewModel.sections.get().isEmpty)
}
```

Let’s update it to expect it to publish `loading` instead:

```swift
// MenuList.ViewModelTests
// ...
func testWhenFetchingStartsPublishesLoading() throws {
    let viewModel = MenuList.ViewModel(menuFetching: MenuFetchingStub(returning: .success([])))

    switch viewModel.sections {
    case .loading: break
    case _: XCTFail("Expected .loading, got \(viewModel.sections)")
    }
}
```

The new test fails with:

```swift
❌ failed - Expected .loading, got success([])
```

## Step 3: Make the test pass

To make the test pass, we need to set `sections = .loading` before starting the menu request.

```swift
// MenuList.ViewModel.swift
// ...

// Set the state to `.loading` before starting a fetch
sections = .loading

menuFetching.fetchMenu()
    // ...
```

This change alone is enough to make the test pass, but I would recommend also updating the default value set for `sections`:

```swift
// MenuList.ViewModel.swift
// ...
@Published private(set) var sections: RemoteData<[MenuSection], Error> = .notAsked
```

Because we call `fetchMenu()` immediately after `init`, the first value published for `sections` will always be `loading`, but defining it as `notAsked` by default future-proofs the code to be in a consistent state in case we’ll move the fetch start to later on in the life-cycle.

## Step 4: Update the view

As our `TODO` comment reminds us, now that the ViewModel correctly models the state, it’s time to show the user helpful information about it.

Eventually, I’d like to render an animation that engages the user, improving the app’s *perceived performance* while the data is. For the moment, “Loading…” will do the job just fine. This is another application of the Partition Problem and Solve Sequentially principle from the book. Handling the state and handling it with a satisfying UX are two different problems. We can address them individually, one after the other.

Test-Driven Development puts emphasis on moving in small iterations and solving one problem at a time. Showing “Loading…” is a barebone, underwhelming UX, but a correct one nonetheless. Committing this change leaves the app in a working state. In fact, we could even ship the app as it is, to production or to the testers, and gather real-world feedback while refining how we show the user that the data is.

```swift
// MenuList.swift
// ...
var body: some View {
    switch viewModel.sections {
    case .success(let sections):
        // ...
    case .failure(let error):
        // ...
    case .loading, .notAsked:
        // Because we start the request when the screen loads, the user will never be left in
        // the .notAsked state, so we can group it with the .loading state.
        Text("Loading...")
    }
}
```

## Step 4: Final manual test

I’m confident about the implementation as it is. The Swift type system and compiler ensure the state’s consistency and, at runtime, Combine and SwiftUI handle all the data synchronization and rendering.

Still, as much as I trust the setup, it doesn’t hurt to give it a run-through manually.

To help with that, we can force a delay in the delivery of the menu sections, so our slow brain has the time to look at the app and check it works as expected.

Combine offers a delay(for:tolerance:scheduler:options:) method on `Publisher` that we can use for this.

```swift
// MenuList.ViewModel.swift
// ...
menuFetching
    .fetchMenu()
    .map(menuGrouping)
    .delay(for: 2, scheduler: RunLoop.main)
    .sink(/* ... */)
```

<figure class="aligncenter is-resized">
<img src="https://web.archive.org/web/20230201181615im_/https://i0.wp.com/tddinswift.com/wp-content/uploads/2021/07/albertos-loading-demo.gif?resize=193%2C422&amp;ssl=1" class="jetpack-lazy-image" width="193" height="422" alt="GIF showing the app launching, the view, and finally the loaded menu" />
<img src="https://web.archive.org/web/20230201181615im_/https://i0.wp.com/tddinswift.com/wp-content/uploads/2021/07/albertos-loading-demo.gif?resize=193%2C422&amp;ssl=1" width="193" height="422" alt="GIF showing the app launching, the view, and finally the loaded menu" />
</figure>

Fun fact. When working on this code, I added the delay call, verified it worked, and then took a break. Once back on it, I run the unit tests, *as one does when picking up a codebase after a break*, and they failed. I spent a good 5 minutes scratching my head trying to figure out why and eventually realized the timeout in the tests was 1 second, and I had the delay set to 2 seconds.

To avoid being a victim of my forgetfulness again in the future, I wrote a custom delay implementation that only runs in debug and not as part of the unit tests:

```swift
// Publisher+DebugDelay.swift
import Combine
import Foundation

extension Publisher {

    /// In DEBUG, delay the publishing by the given `interval`. In other build configurations or
    /// when running the tests, discard.
    func debugDelayOnMainThread(for interval: RunLoop.SchedulerTimeType.Stride) -> Publishers.Delay<Self, RunLoop> {
        var computedInterval: RunLoop.SchedulerTimeType.Stride = 0
        #if DEBUG
        computedInterval = NSClassFromString("XCTestCase") == nil ? interval : 0
        #endif

        return delay(for: computedInterval, scheduler: RunLoop.main)
    }
}
```
