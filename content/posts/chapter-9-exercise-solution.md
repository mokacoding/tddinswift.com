---
title: 'Chapter 9 Exercise Solution'
date: 2021-07-18T21:26:58+00:00
description: "Chapter 9: Testing JSON Decoding – Exercise Solution Swift’s Decodable provides a native, robust, and flexible way to parse JSON data into a domain object. Gone are the days where we’d …"
slug: chapter-9-exercise-solution
---

Swift’s `Decodable` provides a native, robust, and flexible way to parse JSON data into a domain object. Gone are the days where we’d see a new open-source Swift JSON parser published every week.

As we discussed in Chapter 9, `Decodable` is so powerful that, in many circumstances, it makes writing unit tests and Test-Driven Development unnecessary. That chapter’s exercise suggests imagining the backend responds with JSON data in an object rather than in a flat array.

```swift
{
    "items": [
        {
            "name": "spaghetti carbonara",
            "category": "pasta",
            "spicy": false,
            "price": 5.0
        },
        {
            "name": "penne all'arrabbiata",
            "category": "pasta",
            "spicy": true,
            "price": 5.5
        }
    ]
}
```

“How would you decode `MenuItem` if the response was an object?” asks the book. The answer is somehow underwhelming: `MenuItem` remains the same. We merely need a new `Decodable` object to represent the object the API responds with.

That there’s nothing fancier than adding a sort of Data Transfer Object is a treat in and of itself, though, yet another example of how straightforward JSON can be in Swift.

The test one would write for this object’s has the same structure as those we wrote in the chapter. While looking at it in this bonus content post, I want to discuss something I didn’t touch on in the book: **you can iterate on the tests the same way you iterate on the code.** You can start with a broad strokes test, make it pass, then progressively make its assertions shaper, adjusting the production code accordingly.

Let’s imagine this change of API response comes after you already wrote all the for `MenuItem`. How would we go about updating the software to work with it?

It all starts with a test for the new response:

```swift
// MenuResponseTests.swift
@testable import Albertos
import XCTest

class MenuResponseTests: XCTestCase {

    func testWhenDecodedFromJSONDataHasArrayOfValidMenuItems() throws {
        // Arrange
        let json = #"""
{
    "items": [
        {
            "name": "spaghetti carbonara",
            "category": "pasta",
            "spicy": false,
            "price": 5.0
        },
        {
            "name": "penne all'arrabbiata",
            "category": "pasta",
            "spicy": true,
            "price": 5.5
        }
    ]
}
"""#
        let data = try XCTUnwrap(json.data(using: .utf8))

        // Act
        // ???

        // Assert
        // ???
    }
}
```

*Note that I wouldn’t have those Arrange, Act, Assert comments in the actual test code. I’m using them here to assist in the explanation.*

```swift
// MenuResponse.swift

struct MenuResponse: Decodable {}
```

What kind of assertion or assertions can we write? At this point, `MenuResponse` is but a placeholder implementation. Before moving on with the actual, I want to ensure all the scaffolding is in place, so I’ll just ensure that the doesn’t throw. *Take small steps.*

```swift
// Act + Assert
XCTAssertNoThrow(try JSONDecoder().decode(MenuResponse.self, from: data))
```

This test passes. I might actually commit the code as it is, with a title like “Add `MenuResponse` `Decodable` type – Empty for the moment”.

Next, how can we get help from the test to ensure the of the items from the JSON takes place? We can start by ensuring that `MenuResponse` has a `items: [MenuItem]` property with two items.

```swift
// Act
let response = try JSONDecoder().decode(MenuResponse.self, from: data)

// Assert
XCTAssertEqual(response.items.count, 2)
    // Compiler says: ❌ Value of type 'MenuResponse' has no member 'items'
```

Writing this assertion brings up a compiler failure because `MenuResponse` is still nothing but a placeholder. Let’s fix that:

```swift
// MenuResponse.swift
struct MenuResponse: Decodable {

    let items: [MenuItem]
}
```

The test compiles and passes, too, because we’ve done all the work to configure `MenuItem`‘s in the book.

What now? Well, if I were a bit paranoid, I’d want to ensure that the elements that go into `items` are actually decoded from the input JSON:

```swift
let response = try JSONDecoder().decode(MenuResponse.self, from: data)

XCTAssertEqual(response.items.count, 2)

let firstItem = try XCTUnwrap(response.items.first)
XCTAssertEqual(firstItem.name, "spaghetti carbonara")
XCTAssertEqual(firstItem.category, "pasta")
XCTAssertEqual(firstItem.spicy, false)
XCTAssertEqual(firstItem.price, 5.0)

let secondItem = try XCTUnwrap(response.items.last)
XCTAssertEqual(secondItem.name, "penne all'arrabbiata")
XCTAssertEqual(secondItem.category, "pasta")
XCTAssertEqual(secondItem.spicy, true)
XCTAssertEqual(secondItem.price, 5.5)
```

This test, too, passes already. Whenever a new test passes out of the gate, we need to take a moment and ask *why*. In this case, it’s because we already implemented the `MenuItem` in Chapter 9.

That the logic to decode individual `MenuItem`s had already been implemented also means that it had already been tested. This brings up a question: are the assertions above redundant?

I think they are. After all, they are the same as what’s in `MenuItemTests`:

```swift
// MenuItemTests.swift
// ...
func testWhenDecodedFromJSONDataHasAllTheInputPropertiesExample1() throws {
    let json = #"{ "name": "a name", "category": "a category", "spicy": true, "price": 1.0 }"#
    let data = try XCTUnwrap(json.data(using: .utf8))

    let item = try JSONDecoder().decode(MenuItem.self, from: data)

    XCTAssertEqual(item.name, "a name")
    XCTAssertEqual(item.category, "a category")
    XCTAssertEqual(item.spicy, true)
    XCTAssertEqual(item.price, 1.0)
}
```

What should we do then? Well, like always, there is no right or wrong answer. It’s a matter of tradeoffs and, to a certain extent, personal taste.

We could keep the redundant assertions. Running a few extra assertions has a negligible runtime overhead compared to the peace of mind of knowing that each test is as thorough as it can be.

There’s also an argument for removing the assertions. While it’s true that the cost is negligible, doing unnecessary work is not something we should endorse. Over time, little tidbits of unnecessary work, each inconsequential on its own, can compound to a noticeable slow down in the test suite run time.

We could go back to only checking `items` count or have a middle ground solution where we check only one property, just to make sure order is preserved:

```swift
XCTAssertEqual(response.items.count, 2)
XCTAssertEqual(response.items.first?.name, "spaghetti carbonara")
XCTAssertEqual(response.items.last?.name, "penne all'arrabbiata")
```

Yet anther option is to fold `MenuItemTests` into `MenuResponseTests`.\
We can keep the fine-grained assertions in the test for the response and delete the redundant test for the individual `MenuItem`.

Software is *soft*! It’s malleable and easy to change. That applies to production software but also to its tests.

We can iterate on the type, style, and granularity of the assertions we use in our tests the same way we would with the implementation details of a piece of production code.

As the codebase evolves, so do its tests. Some tests may become redundant because those for a new component implicitly assert the same behavior. At that point, we can decide to delete them.

Practicing Test-Driven Development, Partition Problem and Solve Sequentially, and moving in small steps, each leaving the codebase building and the tests passing, gives you the freedom and safety to iterate and experiment.
