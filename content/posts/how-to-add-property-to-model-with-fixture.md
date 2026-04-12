---
title: 'Chapter 5 – Exercise Solution'
date: 2021-07-11T19:53:46+00:00
description: "Chapter 5: Changing Tests with Fixtures – Exercise Solution In Chapter 5, we learned about the Fixture Extension technique.Defining a .fixture() method to initialize our domain types in the unit te…"
slug: how-to-add-property-to-model-with-fixture
---

In Chapter 5, we learned about the Fixture Extension technique.\
Defining a `.fixture()` method to initialize our domain types in the unit tests with reasonable default values gave us two significant benefits compared to calling `init` directly. First, we insulated ourselves from having to update all the tests that use a type every time its `init` changes. Second, we gained the ability to write tests that are free from the noise produced by initialization parameters that are necessary but do not affect the behavior under test.

The exercise at the end of Chapter 5 asked to add a `price` property of type `Double` to the `MenuItem` model.

Let’s see the process step by step.

### Step 1: Modify the production code

In Chapter 3, we learned that the compiler is as much part of the Test-Driven Development process as the tests are.\
The idea of starting from a test, which is at the core of TDD, really means starting with a *source of feedback* to drive the change in the code we want to make.\
The source of feedback can be a failing test but also a compiler error.\
When making a change to the type-signature of our code, leveraging the compiler as the source of feedback is a way to get moving fast.

Let’s update `MenuItem` by adding the new property and see what the compiler has to say about it.

```swift
// MenuItem.swift
struct MenuItem {

    // ...
    let price: Double
}
```

If you try to build the production code with the keyboard shortcut `Cmd B`, you’ll see that the build fails.

### Step 2: Make the production code compile

Adding a new property to `MenuItem` implicitly changed the `init` method that the compiler synthesizes for us.\
It went from:

```swift
init(category: String, name: String, spicy: Bool)
```

to:

```swift
init(category: String, name: String, spicy: Bool, price: Double)
```

As such, all the calls to `MenuItem.init` are now failing to compile:

```swift
// AlbertosApp.swift
// ...

// In this first iteration, the menu is a hard-coded array
let menu = [
    MenuItem(category: "starters", name: "Caprese Salad", spicy: false),
      // Compiler says: Missing argument for parameter 'price' in call
    MenuItem(category: "starters", name: "Arancini Balls", spicy: false),
      // Compiler says: Missing argument for parameter 'price' in call
    //...
]
```

To make the production code compile, we need to update the `MenuItem.init` calls in the placeholder dummy `menu`.

```swift
// AlbertosApp.swift
// ...

// In this first iteration, the menu is a hard-coded array
let menu = [
    MenuItem(category: "starters", name: "Caprese Salad", spicy: false, price: 3.5),
    MenuItem(category: "starters", name: "Arancini Balls", spicy: false, price: 4.0),
    //...
]
```

The production code now builds successfully.\
If you try to build the test target, too, using the `Shift Cmd U` keyboard shortcut, you’ll get one compilation error.

### Step 3: Make the test code compile

In the same way as the `MenuItem.init` calls in the production code failed to compile because they missed the new `price` property, so do those in the test code.\
The only difference is that, in the tests, we centralized all of the instantiation logic in our flexible `fixture` static method, so, regardless of how many times we create `MenuItem`s in the tests, we only have one place to update now.

From:

```swift
// MenuItem+Fixture.swift
@testable import Albertos

extension MenuItem {

  static func fixture(
      category: String = "category",
      name: String = "name",
      spicy: Bool = false
  ) -> MenuItem {
      MenuItem(category: category, name: name, spicy: spicy)
          // Compiler says: Missing argument for parameter 'price' in call
  }
}
```

to:

```swift
// MenuItem+Fixture.swift
@testable import Albertos

extension MenuItem {

  static func fixture(
      category: String = "category",
      name: String = "name",
      spicy: Bool = false,
      price: Double = 1.0
  ) -> MenuItem {
      MenuItem(category: category, name: name, spicy: spicy, price: price)
  }
}
```

If you try to build the production and test code now, the operation will succeed.\
The tests will also still pass, as you can verify by running them with the `Cmd U` keyboard shortcut.

Alongside what we discussed in Chapter 5 itself, this exercise should drive home the value of using the Fixture Extension technique in your tests.

In the production code, when we added the `price` property, we got eight compiler errors because we instantiated eight dummy `MenuItem` models in our placeholder menu.\
In the tests, though, despite having a total of six `MenuItem` instances in use, we only got one compiler error.
