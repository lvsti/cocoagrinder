---
layout: post
title:  "Lightweight Object Mocking in Swift"
date:   2017-01-06 19:12:35 +0100
excerpt: "Unit testing in Swift is not as smooth as it was in the ObjC times: creating mock objects without writing a lot of boilerplate is a challenge in itself. Can we make it suck less?"
---

Type-safety in Swift is great but comes at a price: the language poses severe restrictions on dynamic constructs that are not known at compile time. Everything must typecheck, otherwise it doesn't compile. One of the areas where this is especially painful is the world of unit testing. In Objective-C there are countless frameworks (e.g. [Kiwi](https://github.com/kiwi-bdd/Kiwi), [OCMockito](https://github.com/jonreid/OCMockito)) that rely on the dynamic nature of the runtime to summon test doubles, mock objects, swizzle methods, thus easing the process of writing tests. Swift's strict type system renders these solutions unusable. 

### State of the union

So far, the best workaround the community has been able to come up with is to use protocols when declaring dependencies of a type. For example, instead of:

```swift
class Fetcher {
    func fetch() -> String { ... }
}
class Parser {
    func parse(_ string: String) -> [Int] { ... }
}

class MyClass {
    let fetcher: Fetcher
    let parser: Parser
    
    init(fetcher: Fetcher, parser: Parser) {
        ...
    }
}
```

you'd write:

```swift
protocol FetcherProtocol {
    func fetch() -> String
}
class Fetcher: FetcherProtocol {
    func fetch() -> String { ... }
}

protocol ParserProtocol {
    func parse(_ string: String) -> [Int]
}
class Parser: ParserProtocol {
    func parse(_ string: String) -> [Int] { ... }
}

class MyClass {
    let fetcher: FetcherProtocol
    let parser: ParserProtocol
    
    init(fetcher: FetcherProtocol, parser: ParserProtocol) {
        ...
    }
}
```

This way, when you are testing `MyClass`, instead of using a real `Fetcher` and `Parser` instance you can inject any object that conforms to the respective protocol, and by "any" I do mean any, including one that e.g. logs the invocations of a method:

```swift
class MockParser: ParserProtocol {
    // collecting the arguments our method has been called with
    var parseInvocations: [String] = []
    
    func parse(_ string: String) -> [Int] {
        parseInvocations.append(string)
        return [] // some bogus return value
    }
}
```

Now you can set expectations on `parseInvocations` in your favorite UT framework, e.g.:

```swift
// assert that parse(_:) has been called once
expect(parser.parseInvocations.count) == 1
```

This idea is as old as time itself and has been a common practice in a bunch of other languages, but thanks to ObjC being a much more dynamic language, the Cocoa crowd hasn't had to resort to it. Until now.

The above might be a good choice for classes with a narrow interface, but throw at it some more methods and it will be clear that it scales terribly. When you have a mock with N functions all of whose invocations you want to monitor, you'll end up maintaining N arrays. Even worse, what if you want the mock to respond differently from one test case to another? Implement multiple mocks? Or set a flag on the mock that controls the return value of a certain method? For all methods? That is a hell of a lot of boilerplate to write. In ObjC, Kiwi provided all of this out-of-the-box, for free. So what can we do about it in Swift?

### Cuckoo

If you are the lazy kind, you can check out the [Cuckoo](https://github.com/SwiftKit/Cuckoo) framework that promises to generate all the mocks for you by parsing the source files you specify. To be honest, I haven't tried it because I didn't like the DSL they implemented, and there are bits of it that overlap in functionality with [Quick+Nimble](https://github.com/Quick/Quick) which I ubiquitously use, so I leave that as an exercise for the reader.

### MockFive

[MockFive](https://github.com/DeliciousRaspberryPi/MockFive) has taken another approach: it is still you who write the mocks but the (micro)framework provides you with some baked-in functionality to reduce the pain.

Let's see what our `MockParser` implementation would look like with MockFive:

```swift
// 1. implement `Mock`
class MockParser: ParserProtocol, Mock {
    // 2. add necessary boilerplate
    let mockFiveLock = lock()

    func parse(_ string: String) -> [Int] {
        // 3. forward invocations through a call to `stub`
        return stub(identifier: "parse", arguments: string) { _ in 
            // 4. put return value in the closure
            [] 
        }
    }
}
```

As you can see, each time the subject under test calls `parse(_:)` on our mock, the call eventually gets registered in `stub`. The `Mock` protocol exposes an `invocations` array per mock object that collects  these invocations, and this is what you can then use in expectations.

I liked MockFive because it's lightweight and unintrusive, but it's got some unresolved issues:

- the API is confusing at places
- invocations are stringified that makes them hard to test
- stubs are identified by a string, prone to typos
- no DSL for expectations
- last commit happened a year ago, no Swift 3 support

So I decided to pimp it a bit. Meet MockSix.

### MockSix

#### API
Let's look at the API first. To register an invocation: `stub()`, to override a method: `registerStub()`... *\*scratches head\** I wanted to bring this closer to the Kiwi DSL, so I tweaked the function names to reflect what they actually do:

```swift
protocol Mock {
    func registerInvocation<T>(identifier: String, function: String, args: Any?..., andReturn value: T) -> T
    func stub<T>(_ identifier: String, andReturn value: T)
    ...
}

// in the mock
class MockParser: ParserProtocol, Mock {
    let mockSixLock = lock()

    func parse(_ string: String) -> [Int] {
        return registerInvocation(identifier: "parse", 
                                  args: string, 
                                  andReturn: [])
    }
}

// in the test case
let parser = MockParser()
parser.stub("parse", andReturn: [42])   // parse will now return [42]
```

#### Stub identifiers

Next up: stub identifiers (the `"parse"` strings in the above example). Why do we need them at all? My first idea was to drop the identifiers altogether and use the function name instead since function names are unique, right? Right? WRONG. Consider the following class:

```swift
struct Sum {
    func add(_ value: Int) { print(#function) }
    func add(_ value: Double) { print(#function) }
}

let s = Sum()
s.add(5)        // prints "add(_:)"
s.add(5.0)      // surprise! also prints "add(_:)"
```

I set out to search for a way the compiler would return a different value for these two functions but I didn't manage to find one. Alright, user-specified identifiers are a must then. But I still didn't like the idea of having to type these strings twice-- once in the mock and once in the expectation. To eliminate the risk of typos, I decided to convert stub identifiers to enums. I also required implementors of `Mock` to define a `RawRepresentable` type (typically an enum) to identify methods of the mock:

```swift
protocol Mock {
    associatedtype MockMethod : RawRepresentable
    
    func registerInvocation<T>(for method: MockMethod, function: String, args: Any?..., andReturn value: T) -> T
    func stub<T>(_ method: MockMethod, andReturn value: T)
    ...
}

// in the mock
class MockParser: ParserProtocol, Mock {
    enum Methods: Int {
        case parse
    }    
    typealias MockMethod = Methods
    
    let mockSixLock = lock()

    func parse(_ string: String) -> [Int] {
        return registerInvocation(for: .parse, 
                                  args: string, 
                                  andReturn: [])
    }
}

// in the test case
let parser = MockParser()
parser.stub(.parse, andReturn: [42])
```

The `associatedtype` constraint magically binds `registerInvocation` and `stub` together, making it impossible both to stub and to accidentally register invocations for a method that's not present in the enum. The type system is now on our side.

#### Stringified invocations

This was an easy fix: instead of building a string from the arguments, I just shoved them into an array of `[Any?]` along with the name of the function and the raw method identifier:

```swift
struct MockInvocation {
    let methodID: Int
    let functionName: String
    let args: [Any?]
}

protocol Mock {
    var invocations: [MockInvocation] { get }
    ...
}
```

#### DSL for expectations

The icing on the cake would be to make all the above niceties available in my UT framework of choice, that is, Quick+Nimble. My starting point was, again, the [Kiwi expectation syntax](https://github.com/kiwi-bdd/Kiwi/wiki/Expectations#expectations-interactions-and-messages). I wanted to be able to test not just whether an invocation happens, but also how many times and with what arguments. 

First, let's look at the simplified matcher that doesn't care about arguments.

```swift
import Nimble

func receive<T>(_ method: T.MockMethod, times: Int) -> MatcherFunc<T?>
    where T : Mock, T.MockMethod.RawValue == Int {
    
    return MatcherFunc { actualExpression, _ in
        guard
            let doublyWrappedMock = try? actualExpression.evaluate(),
            let wrappedMock = doublyWrappedMock,
            let mock = wrappedMock
        else {
            fatalError("mock is not implementing Mock")
        }

        // check if there has been an invocation with this ID
        let invs = mock.invocations.filter { $0.methodID == method.rawValue }
        return invs.count == times
    }
}
```

Apart from the onion peeling, there is not much going on. Still, these few lines allow us to write the following expectation:

```swift
let parser = MockParser()
parser.parse("aaa")
parser.parse("bbb")
expect(parser).to(receive(.parse, times: 2))  // succeeds
```

Neat, huh? But why stop here? Let's see how argument checking could be implemented. You'd probably want to write something like this:

```swift
expect(myMock).to(receive(.funcThatTakesTwoArgs, with: ["aaa", 42]))
```

This works fine for simple values but doesn't give room for constructs like Kiwi's `any()` (i.e. to match any value of a certain parameter). So let's use an array of verifier functions instead:

```swift
typealias ArgVerifier = (Any?) -> Bool

func receive<T>(_ method: T.MockMethod, with verifiers: [ArgVerifier]) -> MatcherFunc<T?>
    where T : Mock, T.MockMethod.RawValue == Int {
    
    return MatcherFunc { actualExpression, _ in
        // [unwrapping `mock` omitted for clarity]
        let matching = mock.invocations
            .filter { $0.methodID == method.rawValue && $0.args.count == verifiers.count }
            .filter { inv in
                // check each argument with the corresponding verifier
                for i in 0..<inv.args.count {
                    if !verifiers[i](inv.args[i]) {
                        return false
                    }
                }
                return true
            }
        return matching.count == 1
    }
}
```

You can pass in any suitable function for a verifier but I have collected the most common ones below:

```swift
/// argument matches the given value
func theValue<T : Equatable>(_ val: T) -> ArgVerifier

/// argument is nil
func nilValue() -> ArgVerifier

/// argument matches anything (always succeeds)
func any() -> ArgVerifier

/// argument matches any of the values in the array
func any<T : Equatable>(of options: [T?]) -> ArgVerifier

/// argument makes the predicate true
func any<T>(passing test: @escaping (T) -> Bool) -> ArgVerifier
```

With this at hand, writing complex expectations is a piece of cake:

```swift
expect(myMock).to(receive(.veryComplicatedFunction, with: [
    theValue("aaa"), 
    any(of: [2, 3, 5, 7, 11, 13]),
    any(),
    any { (array: [Int]) in !array.filter({ $0 > 10 }).isEmpty }
]))
```

### Conclusion

Mocking objects in Swift's static type system is nowhere as comfortable as it was in Objective-C, and it will remain so unless the language and the runtime adds some kind of support, as it was the case e.g. with `@testable` imports. Until then, we have to make do with workarounds that either escape the language (Cuckoo) or assume writing boilerplate. Here I presented a small improvement for the latter case. You can find MockSix in [this repo](https://github.com/lvsti/MockSix/) and the Nimble matchers [here](https://github.com/lvsti/NimbleMockSix/).
