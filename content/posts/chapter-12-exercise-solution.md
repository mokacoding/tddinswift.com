---
title: 'Chapter 12 – Exercise Solution'
date: 2021-07-28T20:40:25+00:00
description: "Chapter 12: Testing Side Effects – Exercise Solution After writing a dozen methods using Test-Driven Development, you’ll likely notice that the easier tests to write are those for code that t…"
slug: chapter-12-exercise-solution
---

After writing a dozen methods using Test-Driven Development, you’ll likely notice that the easier tests to write are those for code that takes one or more inputs and returns an output.\
When your code doesn’t depend on anything and doesn’t affect other components in the system, writing tests for it is straightforward.\
Arrange the inputs, act on the System Under Test to produce the output, assert that the output matches your expectations.\
Unfortunately, unless you work in a *purely functional* language like Elm or Haskell, you can’t write all your code in that fashion.\
Sooner or later, you’ll have to deal with some side effects.

[*Test-Driven Development in Swift*](/) shows a helpful tool for testing side effects: the Spy Test Double.\
A Spy implements a dependency from the production code that registers what method calls it receives and which arguments were passed to them.

In the book, we built a Spy for the fictional third-party payment library.\
The companion code includes an analytics library, *Hippo Analytics*, and the book suggests adding events to track events for when the user loads a menu item details screen and for when it orders it.

Let’s write this new code together.\
The process won’t be much different from that in the book, but we’ll have a chance to make some additional considerations about testing events logging in particular.

Hippo Analytics exposes a client object to log events.\
Here’s its public interface:

```swift
class HippoAnalyticsClient {

    init(apiKey: String)

    func logEvent(named name: String, properties: [String: Any]? = .none)
}
```

The first thing to do is define an abstraction around `HippoAnalyticsClient`.\
Tactically speaking, the abstraction will let us write Test Doubles for the dependency to test how our code interacts with it.\
From a design quality point of view, it will also insulate our code from the library’s implementation details.

```swift
// EventLogging.swift
protocol EventLogging {

    func log(name: String, properties: [String: Any])
}
```

```swift
// HippoAnalyticsClient+EventLogging.swift
import HippoAnalytics

extension HippoAnalyticsClient: EventLogging {

    func log(name: String, properties: [String : Any]) {
        logEvent(named: name, properties: properties)
    }
}
```

Having `EventLogging` in place removes the third-party dependency overhead from our working memory.

We want to log two events, for when `MenuItemDetail` renders and when the user adds a `MenuItem` to the order.\
Let’s start with the screen rendered event.

This event directly relates to the view’s life-cycle, making the ViewModel the most appropriate place to log it.

Like we did a few times in the book, let’s use a default value for the dependency in the production code to get us started writing the code using it and get feedback on that implementation first.\
We’ll tackle injecting it properly later on —Partition Problem and Solve Sequentially.

```swift
// MenuItemDetail.ViewModel.swift
// ...
private let eventLogging: EventLogging

// TODO: Remove default value after completing the event logging implementation
init(
    item: MenuItem,
    orderController: OrderController,
    eventLogging: EventLogging = HippoAnalyticsClient(apiKey: "abcd")
) {
    self.item = item
    self.orderController = orderController
    self.eventLogging = eventLogging
    // ...
```

The `eventLoggin` parameter is what Micheal Feathers calls *seam* in Working Effectively with Legacy Code. We can use this seam to insert a Test Double to write tests for the logging behavior.

It might be tempting to log the event in the ViewModel `init` method, but keep in mind that there is no guarantee that the app will initialize a ViewModel instance and render the view using it in direct sequence.\
In fact, if you add a breakpoint or a `print` statement in `MenuItemDetail.ViewModel`‘s `init` method then run the app, you’ll notice that as soon as the menu list loads on the screen, all the ViewModels for each of the visible items initialize.\
*If you know of a way to lazy load them, please get in touch on Twitter at @mokagio or write to [hello@tddinswift.com](#)*.

A better point in the view life-cycle where to log the event is when the view actually renders.\
SwiftUI offers the onAppear(perform:) method on `View` to hook up to that precise life-cycle moment.

```swift
// MenuItemDetail.ViewModelTests.swift
// ...
func testOnAppearLogsMenuItemDetailVisitedEvent() throws {
    let eventLoggingSpy = EventLoggingSpy()
    let item = MenuItem.fixture(name: "item")
    let viewModel = MenuItemDetail.ViewModel(
        item: item,
        orderController: OrderController(order: .fixture()),
        eventLogging: eventLoggingSpy
    )

    viewModel.onAppear()

    XCTAssertEqual(eventLoggingSpy.loggedEvents.count, 1)
    let event = try XCTUnwrap(eventLoggingSpy.loggedEvents.first)
    XCTAssertEqual(event.name, "menu_item_detail_visited")
    XCTAssertEqual(event.properties["item_name"] as? String, "item")
}
```

We can make this test pass by adding a new `onAppear()` method to `MenuItemDetail.ViewModel` which logs the event:

```swift
// MenuItemDetail.ViewModel.swift
func onAppear() {
    eventLogging.log(
        name: "menu_item_detail_visited",
        properties: ["item_name": item.name]
    )
}
```

We *could* do a bit of refactoring and move the implementation details of this event outside of the ViewModel, perhaps in an `EventLogging` protocol extension, but I don’t see much value in that at this point.\
The Refactor step in the Red, Green, Refactor workflow is about asking the question, not necessarily making a refactor.\
Just because there’s a refactor opportunity, it doesn’t mean you should pursue it.\
*Check out this post for a heuristic to help you decided when a refactor is appropriate.*

The approach to implementing logging the event for when the user adds the item to the order is pretty much the same.\
We can keep the logging at the ViewModel level or implement it at the PaymentProcessing level.

I think we should start with the implementation in the ViewModel because we already have the infrastructure there and because logging events seems like a responsibility for which a coordination-type like the ViewModel is better suited than a service type like a concrete `PaymentProcessing` implementation.

```swift
// MenuItemDetail.ViewModelTests.swift
// ...
func testOnAddingItemToOrderLogsMenuItemDetailOrderedEvent() throws {
    let eventLoggingSpy = EventLoggingSpy()
    let viewModel = MenuItemDetail.ViewModel(
        item: .fixture(name: "item"),
        orderController: OrderController(order: .fixture(items: [])),
        eventLogging: eventLoggingSpy
    )

    viewModel.addOrRemoveFromOrder()

    XCTAssertEqual(eventLoggingSpy.loggedEvents.count, 1)
    let event = try XCTUnwrap(eventLoggingSpy.loggedEvents.first)
    XCTAssertEqual(event.name, "menu_item_ordered")
    XCTAssertEqual(event.properties["item_name"] as? String, "item")
}
```

```swift
// MenuItemDetail.ViewModel.swift
// ...
func addOrRemoveFromOrder() {
    if (orderController.order.items.contains { $0 == item }) {
        orderController.removeFromOrder(item: item)
    } else {
        orderController.addToOrder(item: item)
        eventLogging.log(name: "menu_item_ordered", properties: ["item_name": item.name])
    }
}
```

If you, like me, are uncomfortable about the fact that the test above could pass even if we logged the event before the `if` statement in `addOrRemoveFromOrder`, you can add an additional test to ensure that doesn’t happen:

```swift
// MenuItemDetail.ViewModelTests.swift
// ...
func testOnRemovingItemToOrderDoesNotLogEvent() throws {
    let eventLoggingSpy = EventLoggingSpy()
    let item = MenuItem.fixture(name: "item")
    let viewModel = MenuItemDetail.ViewModel(
        item: item,
        orderController: OrderController(order: .fixture(items: [item])),
        eventLogging: eventLoggingSpy
    )

    viewModel.addOrRemoveFromOrder()

    XCTAssertEqual(eventLoggingSpy.loggedEvents.count, 0)
}
```

## Should I test the third-party library direct interaction?

One could argue that the tests we just wrote don’t guarantee that the production code will log the events correctly because they don’t exercise the code that calls the third-party analytics library.

It should come as no surprise that, to test how we call `HippoAnalyticsClient` in the `EventLogging` implementation, we can use Dependency Inversion and the Spy Test Double.

First, let’s define a `protocol` mapping `HippoAnalyticsClient`‘s interface one-to-one that will allow us to inject a Test Double:

```swift
protocol HippoAnalyticsClientInterface {

    func logEvent(named name: String, properties: [String: Any]?)
}

// Because `HippoAnalyticsClientInterface` maps `HippoAnalyticsClient` one-to-one,
// `HippoAnalyticsClient` can conform to it with no extra code.
extension HippoAnalyticsClient: HippoAnalyticsClientInterface {}
```

Then, build an `EventLogging` implementation that expects a type conforming to `HippoAnalyticsClientInterface` to use to log events:

```swift
class EventLogger: EventLogging {

    private let hippoAnalyticsClient: HippoAnalyticsClientInterface

    init(hippoAnalyticsClient: HippoAnalyticsClientInterface) {
        self.hippoAnalyticsClient = hippoAnalyticsClient
    }

    func log(name: String, properties: [String : Any]) {
        hippoAnalyticsClient.logEvent(named: name, properties: properties)
    }
}
```

Finally, construct a Spy for `HippoAnalyticsClientInterface` and use it in the tests:

```swift
class HippoAnalyticsClientInterfaceSpy: HippoAnalyticsClientInterface {

    struct Event {
        let name: String
        let properties: [String: Any]?
    }

    private(set) var loggedEvents: [Event] = []

    func logEvent(named name: String, properties: [String : Any]?) {
        loggedEvents.append(Event(name: name, properties: properties))
    }
}
```

```swift
class EventLoggerTests: XCTestCase {

    func testLogsEventsWithCorrectParameters() throws {
        let hippoAnalyticsSpy = HippoAnalyticsClientInterfaceSpy()
        let eventLogger = EventLogger(hippoAnalyticsClient: hippoAnalyticsSpy)

        eventLogger.log(name: "test", properties: ["key": "value"])

        XCTAssertEqual(hippoAnalyticsSpy.loggedEvents.count, 1)
        let event = try XCTUnwrap(hippoAnalyticsSpy.loggedEvents.first)
        XCTAssertEqual(event.name, "test")
        XCTAssertEqual(event.properties?["key"] as? String, "value")

    }
}
```

I’m on the fence about this approach.

Yes, we gained extra confidence in the code that logs events through the 3rd party library, but was that code that complex and fragile to grant the extra work we put into it?

Also, `HippoAnalyticsClientInterface` is a service protocol built only to allow building Test Doubles on top of it.\
The abstractions we introduced in the app throughout the book, such as NetworkFetching, `PaymentProcessing`, and, in this example, `EventLogging`, all added concrete value in the modeling by restricting the API surface to one tailored for the app’s business logic domain.\
`HippoAnalyticsClientInterface` doesn’t add any value to the production code.

I wouldn’t recommend going down this path unless you necessitate extra confidence in the event logging implementation.\
An example I can think of this are big codebases touched by many developers.\
The more people have access to the code, the more likely a bug can sneak into it, so it might be good to err on the side of caution.\
Even then, though, it might be better to approach the problem differently and isolate the logging implementation in a dedicated first-party library that feature-developers can utilize.\
This way, changes to the lower level logic calling the third-party service would be easier to track, and the need for the extra level of design complexity would disappear.

------------------------------------------------------------------------

Event logging is a crucial part of any software project.\
One that, unfortunately, is too often neglected because of perceived challenges in testing it.\
This post showed testing analytics code is more feasible than it seems.\
By combining the Dependency Inversion Principle with a Spy Test Double, you can add tests to most of your event logging implementation.
