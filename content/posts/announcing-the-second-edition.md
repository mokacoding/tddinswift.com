---
title: "Test-Driven Development in Swift, Second Edition is here"
date: 2026-06-11
description: "Fully revised second edition: with Swift Testing and structured concurrency."
slug: announcing-the-second-edition
draft: false
---

The second edition of [_Test-Driven Development in Swift_](https://link.springer.com/book/10.1007/979-8-8688-2637-5) is finally out.

When the first edition shipped in 2021, we wrote tests with XCTest and asynchronous code with Combine.
But the Swift landscape is always evolving and improving.
Between [Swift Testing](https://developer.apple.com/xcode/swift-testing/) and [async await](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/), the way we write and test code has changed.
This edition is a full revision to match.

## What's new

**Swift Testing throughout.**
Every test in the book is now written with Swift Testing and making the most of its natural language support.
But fear not, if your codebase still runs on XCTest, a new appendix holds a completed XCTest reference, so you can bridge the book's lessons to your older tests.

**Structured concurrency.**
The networking chapters have been rewritten with `async`/`await`.
The old `AnyPublisher<T, Error>` signatures are now `async throws -> T`, and the tests await results directly instead of using cumbersome `XCTestExpectation` or `sink`.

**A refreshed sample app.**
Alberto's menu-ordering app now uses the latest SwiftUI APIs and look and feel.
Chapter 7 keeps `@Published`/`ObservableObject` as the teaching path and adds a sidebar on where `@Observable` fits.

## Get it

Grab a copy from <a href="https://link.springer.com/book/10.1007/979-8-8688-2637-5" target="_blank" rel="noopener">Apress</a> or <a href="https://www.amazon.com/dp/B0GLFXWGLZ" target="_blank" rel="noopener">Amazon</a>, and [pick up the free bonus material](/gift/) while you're here.

## If you already own the first edition

The Test-Driven Development *philosophy* hasn't changed, because it was never tied to a framework.
The menu-ordering app, the Test Doubles (Stub, Spy, Fake, Dummy), and the red/green/refactor rhythm are all still here.

What changed is the *how*: Swift Testing, structured concurrency, modern SwiftUI.
If you're moving a codebase onto those, the second edition is a guided tour of the new patterns in context.

Questions about the new edition? Find me on [X](https://x.com/mokagio) or at [gio@mokacoding.com](mailto:gio+tddinswift@mokacoding.com).
