---
title: Migrating your test suite from XCTest to Swift Testing
date: 2026-01-01
draft: true
edition_compatibility: ["v1", "v2"]
---

TODO: Practical migration guide for v1 readers who own existing test suites
in XCTest and want to translate them. Cover:

- The mental model shift (class → struct, lifecycle, parallelism)
- Mechanical translations table
- Edge cases where Swift Testing has no direct XCTest equivalent
- Edge cases where XCTest has no Swift Testing equivalent (yet)
- When to keep XCTest (UI tests, performance tests as of 2026)
- How to run mixed test bundles
