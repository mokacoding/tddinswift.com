---
title: 'Chapter 10 – Exercises Solution – Part 3'
date: 2021-07-23T20:48:35+00:00
description: "Chapter 10 – Testing Network Code – Exercise Solution Part 3: How to keep code tidy when implementing multiple API endpoint The Test-Driven Development in Swift chapter on using TDD with networking…"
slug: chapter-10-exercises-solution-part-3
---

## Part 3: How to keep code tidy when implementing multiple API endpoint

The [*Test-Driven Development in Swift*](/) chapter on using TDD with networking code closes with two exercise suggestions.\
In the previous two bonus content posts, we saw how to make the base URL used by `MenuFetcher`, the component initiating network requests, configurable at `init` time by writing a test [with a Stub](/chapter-10-exercise-solution-part-1/) or [with a Spy](/chapter-10-exercise-solution-part-2/).

The other exercise asks to add data fetching from the dish of the day API endpoint, which returns a single JSON object representing a dish for the menu ordering app we built throughout the book to render.

I suggest adding a new endpoint to solidify the learnings from the chapter because the steps to take are the same as shown in the chapter.\
A walkthrough of that process would be nothing more than a repetition of what’s already in the book.\
Instead, I want to make this issue different by discussing a few trade-offs and geeking out on a DRY networking implementation.\
We’ll discuss how to approach the abstraction layer that represents the capability to fetch the dish of the day, how to structure its concrete implementation, then play around with Swift generics to remove boilerplate code.

## Multiple abstractions or one to rule them all?

The book teaches TDD by building an approximation of a real-world application for a fictional Italian restaurant owner called Alberto.\
Alberto’s menu ordering app fetches the menu from an API, renders it on screen, and lets users order dishes.

The codebase defines an abstraction for the component in charge to fetch the menu:

```swift
// MenuFetching.swift
import Combine

protocol MenuFetching {

    func fetchMenu() -> AnyPublisher<[MenuItem], Error>
}
```

The abstraction is valuable in the production code because it insulates architectural layers from implementation details.\
The tests can use `MenuFetching` to build Test Doubles to control different behavioral facets and cover a broader set of scenarios.

When it comes time to add a new endpoint with data to fetch, we have two options: add a new method to the existing `protocol` or create a new dedicated abstraction.

If we were to add a new method to `MenuFetching` it would be appropriate to rename it to reflect it’s now representing more than a single resource fetch.

```swift
protocol ResourceFetching {

    func fetchMenu() -> AnyPublisher<[MenuItem], Error>

    func fetchDishOfTheDay() -> AnyPublisher<MenuItem, Error>
}
```

Alternatively, we could leave `MenuFetching` as it is and add a new dedicated abstraction:

```swift
protocol MenuItemFetching {

    func fetchDishOfTheDay() -> AnyPublisher<MenuItem, Error>
}
```

I decided to name it `MenuItemFetching` to represent the type of resource it procures.\
`DishOfTheDayFetching` is another possible name we could have used.

I prefer the dedicated abstraction approach to the general one.\
A sharp, narrow-focused abstraction allows us to write consuming code that is straightforward to reason about.\
It also declares intent more clearly.

With the abstraction locked in, it’s time to move to the concrete implementation.

## A dedicated implementation or a new method on an existing one?

Coding a concrete implementation for `DishOfTheDayFetching` presents the same design dilemma as choosing how to define the `protocol`.\
Do we write a dedicated `DishOfTheDayFetcher` implementation or make our existing networking component, `MenuFetcher`, conform to `DishOfTheDayFetching` as well as `MenuFetching`?

Building a dedicated object to implement `DishOfTheDayFetching` would keep the code homogeneous.\
We’d have `MenuFetcher` for `MenuItemFetching` and `DishOfTheDayFetching` for `DishOfTheDayFetcher`.\
Symmetric, straightforward, and satisfying.

But also *shortsighted*.

Spawning a dedicated object for each `protocol` in the codebase is a lost opportunity to leverage a sort of economy of scale in our codebase componentry.

If we give a single object the responsibility for multiple resource fetches, we can optimize its implementation for better code reuse and extensibility.

Let’s not try to be too smart, though.\
We shall write a straightforward implementation first, and only once that works, attempt to optimize it.\
*Partition problem and solve sequentially*.\
Don’t try to do two things at once.

I’ll jump through the TDD process and only show the finished production code here.

```swift
// APIClient.swift
class APIClient: DishOfTheDayFetching, MenuFetching {

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

    func fetchDishOfTheDay() -> AnyPublisher<MenuItem, Error> {
        let request = URLRequest(url: baseURL.appendingPathComponent("dish_of_the_day.json"))

        return networkFetching.load(request)
            .decode(type: MenuItem.self, decoder: JSONDecoder())
            .eraseToAnyPublisher()
    }
}
```

The dish of the day fetching tests look the same as the ones for the menu list.\
If you don’t have the book at hand, you can see the menu list fetching tests here.

Even though we now have a single object implementing multiple requests, as long as we follow the Dependency Inversion Principle and declare our components as interacting through abstractions, the rest of the application won’t know about it.\
For example, `MenuList.ViewModel` expects a `MenuFetching`-conforming type as an `init` parameter.\
When we pass it an `APIClient` instance, `MenuList.ViewModel` will only have access to the methods `APIClient` implements from `MenuFetching`, not to its entire interface.\
This allows us to keep all the benefits of local reasoning and encapsulation while also unlocking code optimizations in the `APIClient` implementation.

## Refactor: DRY

Looking at the `APIClient` implementation from a distance, we can see both methods follow the same pattern:

```swift
let request = URLRequest(url: baseURL.appendingPathComponent(<# path #>))

return networkFetching.load(request)
    .decode(type: <# resource type #>.self, decoder: JSONDecoder())
    .eraseToAnyPublisher()
```

Let’s apply “Don’t Repeat Yourself” and extract this duplication in a dedicated method:

```swift
// APIClient.swift
// ...
func fetchMenu() -> AnyPublisher<[MenuItem], Error> {
    return fetch(fromPath: "menu_response.json")
}

func fetchDishOfTheDay() -> AnyPublisher<MenuItem, Error> {
    return fetch(fromPath: "dish_of_the_day.json")
}

private func fetch<Resource>(
    fromPath path: String
) -> AnyPublisher<Resource, Error> where Resource: Decodable {
    let request = URLRequest(url: baseURL.appendingPathComponent(path))

    return networkFetching.load(request)
        .decode(type: Resource.self, decoder: JSONDecoder())
        .eraseToAnyPublisher()
}
```

I’d like to take a moment for us to appreciate how neat generics are.\
Thanks to them, we can use one implementation to power two functions with different return types.

Thanks to the extensive unit tests coverage we developed by writing our code test-first, the only thing we need to do to verify this refactor is running the tests.\
Without tests, we’d have to manually trigger both code paths by using the app, which is more time-consuming.

## Refactor: Separate configuration from execution logic

Let’s keep geeking out with the code.

`APIClient` is currently responsible for executing requests and holding the knowledge of the endpoints’ path and the type of resource they return.\
We can extract this knowledge in a dedicated object to make it easier to consult and update:

```swift
// Endpoint.swift
import Foundation

struct Endpoint<Resource: Decodable> {
    let path: String
    let resourceType = T.self

    func urlRequest(with baseURL: URL) -> URLRequest {
        URLRequest(url: baseURL.appendingPathComponent(path))
    }
}
```

`Endpoint` is a value type that we can use to hold the information about the path of an API endpoint and the type of domain object to map its response to.\
We can then build a list of endpoints for our API:

```swift
// Endpoints.swift
enum Endpoints {

    static let menu = Endpoint<[MenuItem]>(path: "menu_response.json")
    static let dishOfTheDay = Endpoint<MenuItem>(path: "dish_of_the_day.json")
}
```

Finally, we can update `APIClient` to use the new `Endpoint` type:

```swift
// APIClient.swift
// ...
func fetchMenu() -> AnyPublisher<[MenuItem], Error> {
    return fetch(from: Endpoints.menu)
}

func fetchDishOfTheDay() -> AnyPublisher<MenuItem, Error> {
    return fetch(from: Endpoints.dishOfTheDay)
}

private func fetch<Resource>(
    from endpoint: Endpoint<Resource>
) -> AnyPublisher<Resource, Error> where Resource: Decodable {
    return networkFetching.load(endpoint.urlRequest(with: baseURL))
        .decode(type: endpoint.resourceType, decoder: decoder)
        .eraseToAnyPublisher()
}
```

Once again, we can get fast feedback on this refactor by running the tests.

Admittedly, for a class the size of `APIClient` in a codebase the size of the app from the book, this refactoring was more of a yack shave.\
But in a bigger codebase, with more API endpoints to interact with, there’s a real benefit in separating the endpoint configuration logic from the code that performs the request itself.\
When the configuration code is isolated, it becomes easier to find, reason about, and update.

------------------------------------------------------------------------

Networking is a core and multifaceted part of most software, so it’s not surprising that it’s taken us three bonus content episodes to discuss two relatively simple exercises.\
We only worked with straightforward GET requests here, but there’s much more to networking than what we’ve covered, like sending data to the backend with POST requests or multipart form uploads.

That’s all to say, networking is a complicated part of application development for which testing is crucial, both to guide the design and ensure the behavior.\
I hope the book and this series of posts have given you good food for thought on how to approach the networking needs of your own software projects.

As always, I’m keen to hear what you think.\
Get in touch on Twitter @mokagio or email me at [hello@tddinswift.com](#).
