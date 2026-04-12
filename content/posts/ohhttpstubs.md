---
title: 'OHHTTPStubs'
date: 2021-08-01T20:57:52+00:00
description: "Network testing with OHHTTPStubs Throughout Test-Driven Development in Swift, we wrote tests using vanilla XCTest because it’s the lowest common denominator for any testing you might do in yo…"
slug: ohhttpstubs
---

Throughout [*Test-Driven Development in Swift*](/), we wrote tests using vanilla XCTest because it’s the lowest common denominator for any testing you might do in your Swift applications. That’s not to say that third-party libraries cannot provide helpful support in practicing TDD. At the end of Chapter 10, where we discussed networking, I mentioned a noteworthy open-source library called OHHTTPStubs.

OHHTTPStubs offers a smooth DSL to stub just about any kind of network request your application might be performing, injecting custom data or error in place of its response.

You can fetch the library using your favorite dependency management system and access it in your code by importing both the `OHHTTPStubs` module and its Swift helpers companion, `OHHTTPStubsSwift`.

In the code from the book, we can test how `MenuFetcher` processes data it receives from the network using OHTTPStubs instead of passing a `NetworkFetchingStub` instance:

```swift
@testable import Albertos
import Combine
import OHHTTPStubs
import OHHTTPStubsSwift
import XCTest

class MenuFetcherOHHTTPStubsTests: XCTestCase {

    var cancellables = Set<AnyCancellable>()

    func testWhenRequestSucceedsPublishesDecodedMenuItems() throws {
        let json = """
[
    { "name": "a name", "category": "a category", "spicy": true, "price": 1.0 },
    { "name": "another name", "category": "a category", "spicy": true, "price": 2.0 }
]
"""
        let data = try XCTUnwrap(json.data(using: .utf8))
        let menuFetcher = MenuFetcher()

        stub(condition: pathEndsWith("menu_response.json")) { _ in
            HTTPStubsResponse(data: data, statusCode: 200, headers: .none)
        }

        let expectation = XCTestExpectation(description: "Publishes decoded [MenuItem]")

        menuFetcher.fetchMenu()
            .sink(
                receiveCompletion: { _ in },
                receiveValue: { items in
                    XCTAssertEqual(items.count, 2)
                    XCTAssertEqual(items.first?.name, "a name")
                    XCTAssertEqual(items.last?.name, "another name")
                    expectation.fulfill()
                }
            )
            .store(in: &cancellables)

        wait(for: [expectation], timeout: 1)
    }
}
```

As you can see, the test is the same, except for how we create the SUT and the addition of the HTTP stub configuration:

```swift
let menuFetcher = MenuFetcher()

stub(condition: pathEndsWith("menu_response.json")) { _ in
    HTTPStubsResponse(data: data, statusCode: 200, headers: .none)
}
```

I won’t go into the details of the DSL, but you can read more about it in the official usage examples.

## OHHTTPStubs use case

OHTTPStubs’ ability to hijack `URLSession` without any extra setup required in the test suite or production code makes it an invaluable ally in *test rescues*. If the code you are dealing with is convoluted and hard to modify, OHTTPStubs can be your mightiest ally to build confidence in the networking code through tests. When it’s dangerous or time-consuming to use Dependency Inversion and Injection, not having to do any extra setup in the production code can be a lifesaver.

On the other hand, I tend not to bring it into my projects when I start from scratch. As powerful and convenient as the library is, I fear being able to seamlessly stub network requests might make me miss good opportunities to improve my design. I find value in abstracting `URLSession` behind a protocol like `NetworkFetching`. Even though `URLSession` is the only concrete implementation my apps will ever use, having a `protocol` tailored to the networking needs of my project reduces the surface area of options accessible to the code interfacing with it, enhancing local reasoning.

If you choose to use OHHTTPStubs in your test suite, it should be because you like the syntax and the ergonomics, not as an excuse to disregard good design in the way you access `URLSession`.
