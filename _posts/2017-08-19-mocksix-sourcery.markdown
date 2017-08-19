---
layout: post
disqus: true
title:  "MockSix + Sourcery: Happily Ever After?"
date:   2017-08-19 11:30:35 +0200
excerpt: "MockSix mocks are compact but still require writing some boilerplate. In this post, I'm setting out to see how much of that could be automated."
---

A couple of months ago I posted about [my take on object mocking](/cocoagrinder/2017/01/06/lightweight-object-mocking-in-swift.html) in Swift and how I was trying to avoid employing frameworks that used code generation techniques. However, real-world experience has recently made me change my mind. In this post, I'm going to give an overview of how I found my way back to painless unit testing which I thought was forever lost with the departure from ObjC.

For a long time, I had a negative bias towards any code generation framework because I basically considered them a hack compared to a pure Swift-based solution and was afraid that they would fail me when I'd least expect it. But it so happened recently that in my day to day work I had to deal with (and even create) some generated APIs and, soon, conjuring code up instead of typing it didn't seem that wild and alien anymore. So I decided to reconsider my stance on generated mocks and see whether it can help improve our team's productivity when it comes to testing.

### MockSix

[MockSix](https://github.com/lvsti/MockSix) is the microframework I developed and wrote about in [my last post](/cocoagrinder/2017/01/06/lightweight-object-mocking-in-swift.html). It's lightweight, strongly typed, and lets you create mock objects for interface protocols. The [NimbleMockSix](https://github.com/lvsti/NimbleMockSix) companion library provides matchers to use with Quick+Nimble for a more pleasant developer experience when writing test expectations. 

```swift
// MockSix example
protocol ParserProtocol {
    func parse(_ string: String) -> [Int]
}

class MockParser: ParserProtocol, Mock {
    enum Methods: Int {
        case parse
    }    
    typealias MockMethod = Methods

    func parse(_ string: String) -> [Int] {
        return registerInvocation(for: .parse, 
                                  args: string, 
                                  andReturn: [])
    }
}

// given
let myMock = MockParser()

// when
myMock.parse("foo")

// then
expect(myMock).to(receive(.parse, with: [any(of: ["bar", "foo"])]))
```

In my job, we had been creating mocks arbitrarily before adopting MockSix; now all the new mocks have the same structure that makes them easier to understand and maintain. But boilerplate still remains boilerplate despite being uniform, and this is where code generation enters the scene.

### Sourcery

[Sourcery](https://github.com/krzysztofzablocki/Sourcery) is a tool that lets you generate code by parsing swift classes in your project and feeding that type information into templates. If you are not familiar with it, its [documentation](https://cdn.rawgit.com/krzysztofzablocki/Sourcery/master/docs/index.html) is pretty solid and there are already [some](https://www.raywenderlich.com/158803/sourcery-tutorial-generating-swift-code-ios) [blogposts](https://miqu.me/blog/2017/05/21/automatic-bridging-from-swift-to-objective-c-using-sourcery/) about it that are worth to check; from now on, I'll assume that you won't freak out when you encounter a Sourcery stencil later in this post.

Sourcery itself comes with an [example](https://cdn.rawgit.com/krzysztofzablocki/Sourcery/master/docs/mocks.html) on mock generation, which hints on this being a common pain all over the Swift developer community. The example follows the naive approach: it uses some extra properties for each method to keep track of invocations. While this is certainly sufficient for demonstration purposes, its features are very limited. 

### MockSix meets Sourcery

By now, you have probably already guessed where all this is going: using Sourcery to generate MockSix-style mocks. So let's dive in!

Given the following protocol:

```swift
protocol FoobarProtocol {
    func doThis(_ string: String) -> [Int]
    func doThat() -> Double
    var myProperty: Int { get }
}
```

let's look at the structure of its mock class. The main ingredients are:

&nbsp;1. Conform to `Mock` besides the actual protocol we are creating the mock for
    
```swift
class MockFoobar: FoobarProtocol, Mock {
```

&nbsp;2. Declare an enum for the methods we want to make available in the mock

```swift
    enum Methods: Int {
        case doThis
        case doThat
    }    
    typealias MockMethod = Methods
```
    
&nbsp;3. Implement the above methods by calling through `registerInvocation`

```swift
    func doThis(_ string: String) -> [Int] {
        return registerInvocation(for: .doThis, 
                                  args: string, 
                                  andReturn: [])
    }
    func doThat() -> Double {
        return registerInvocation(for: .doThat, 
                                  andReturn: 0.0)
    }
```

&nbsp;4. Define any properties mandated by the protocol

```swift
    var myProperty: Int = 0
}
```

Now, provided we have access to the type information of `FoobarProtocol`, we should be able to create a template that generates the above mock class by iterating through the protocol methods and properties. Here is how it may look like:

```swift
// note: "\n" characters omitted for the sake of readability

var result = "" // output buffer

// iterate through all protocols which are annotated as "mockable"
for proto in types.protocols where proto.annotations["mockable"] != nil {
    result += "class " + mockClassName(for: proto)
    result += ": " + proto.name + ", Mock { "

    if !proto.allMethods.isEmpty {
        result += "enum Methods: Int { "
        for method in proto.allMethods where !method.isInitializer {
            result += "case " + methodID(for: method)
        }
        result += "} "
        result += "typealias MockMethod = Methods"
    }
    else {
        // assuming we have conformed Int to RawRepresentable
        result += "typealias MockMethod = Int"
    }
    ...
```

For the method enumeration, the only tricky part is preventing clashes between autogenerated method IDs. I haven't yet found a universal solution but here is a naïve implementation that concatenates the name with the argument labels:

```swift
func methodID(for method: SourceryRuntime.Method) -> String {
    let tail = method.parameters
        .map { param in
            guard 
                let label = param.argumentLabel, 
                label != param.name 
            else {
                return param.name
            }
            
            return label + param.name
        }
        .joined(separator: "_")
    
    return method.callName + (tail.isEmpty ? "" : "_" + tail)
}

// produces `doThis_string` for `doThis(_ string: String)`
```

Anyway, moving on:

```swift
    // generate method stubs
    for method in proto.allMethods where !method.isInitializer {
        for (attrName, attr) in method.attributes {
            result += attr.description
        }
        
        let signature = methodSignature(for: method)
        result += "func " + signature + " { "
        
        if method.returnTypeName.isVoid {
            // invocation logging for void methods
            result += "registerInvocation(for: ." + methodID(for: method)
            if !method.parameters.isEmpty {
                result += ", args: "
                result += method.parameters
                    .map { $0.name }
                    .joined(separator: ", ")
            }
            result += ")"
        }
        else {
            // invocation logging for non-void methods
            result += "return registerInvocation(for: ." + methodID(for: method)
            if !method.parameters.isEmpty {
                result += ", args: "
                result += method.parameters
                    .map { $0.name }
                    .joined(separator: ", ")
            }
            result += ", andReturn: ???"
```

...at which point we'll begin to scratch our head. If we want the mocks to be fully generated, we have to take care of the return value for non-void functions (and initial values for properties). But how? Let's handle the straightforward cases first:

- if the type is an (implicitly unwrapped) optional, return `nil`
- if the type is an array or dictionary, return `[]` or `[:]`, respectively

So far so good, but we are yet to cover named (in other words, not compound) types, closures, and tuples. Let's look at named types. In the example, I put a `0` as the initial value for `myProperty`. Naturally, zero is the first thing that comes to mind in case of numeric types. To go one step further, we should realize that we get the same result just by calling the default initializer (i.e. `Int()`). Using this pattern, if a type has a default initializer, we can easily obtain a default value from it.

##### DummyConstructible

Unfortunately, not all types provide an `init()` without parameters (e.g. enums don't), so we should come up with something that handles these cases as well. My first take was the following protocol:

```swift
protocol DummyConstructible {
    static var dummyValue: Self { get }
}
```

Types that conform to `DummyConstructible` should provide a class property of their own type that returns a dummy instance. How it achieves that, it's not our concern, but let's look at a sample conformance for `String.Encoding`:

```swift
extension String.Encoding: DummyConstructible {
    static let dummyValue: String.Encoding = .utf8
}
```

If we assume that all the types that we may encounter in the mock conform to `DummyConstructible`, we can simply use `.dummyValue` both in the call to `registerInvocation` and for initializing properties:

```swift
    func doThat() -> Double {
        return registerInvocation(for: .doThat, 
                                  andReturn: .dummyValue)
    }
    var myProperty: Int = .dummyValue
```

Let's create a function that determines the dummy value based on the type parameter:

```swift
func dummyValue(for type: Type?, typeName: TypeName) -> String {
    if typeName.isOptional || typeName.isImplicitlyUnwrappedOptional {
        return "nil"
    }
    else if typeName.isArray {
        return "[]"
    }
    else if typeName.isDictionary {
        return "[:]"
    }
    else {
        return ".dummyValue"
    }
}
```

Now we can apply this to our template:

```swift
    ...
        else {
            // invocation logging for non-void methods
            result += "return registerInvocation(for: ." + methodID(for: method)
            if !method.parameters.isEmpty {
                result += ", args: "
                result += method.parameters
                    .map { $0.name }
                    .joined(separator: ", ")
            }
            result += ", andReturn: "
            result += dummyValue(for: method.returnType, typeName: method.returnTypeName)
            result += ")"
        }
        
        result += "}"
    } // for all methods
    
    // generate property definitions
    for variable in proto.allVariables {
        for (attrName, attr) in variable.attributes {
            result += attr.description
        }
        
        result += "var " + variable.name + ": " + variable.typeName.name + " = "
        result += dummyValue(for: variable.type, typeName: variable.typeName)
    }

    result += "}"
} // for all protocols
```

Next, let's look at tuples. Since they are composed of named types, we can resolve them by recursing into their components. We just add the following lines to the `dummyValue` function:

```swift
// in dummyValue(for:typeName:)
...
    else if typeName.isTuple, let tuple = typeName.tuple {
        let list = tuple.elements
            .flatMap { dummyValue(for: $0.type, typeName: $0.typeName) }
            .joined(separator: ", ")
        return "(" + list + ")"
    }
...
```

Finally, let's square away closures. For a meaningful minimal closure we can safely ignore the parameters and just focus on the return value which, not suprisingly, is going to be a dummy:

```swift
// in dummyValue(for:typeName:)
...
    else if typeName.isClosure, let closure = typeName.closure {
        if closure.returnTypeName.isVoid {
            return "{ _ in }"
        }
        else {
            let value = dummyValue(for: closure.returnType, 
                                   typeName: closure.returnTypeName)
            return "{ _ in " + value + " }"
        }
    }
...
```

By now, we seem to have covered all possible types but unfortunately we oversaw a little detail when we bet on `DummyConstructible`: namely, we assumed that we could _construct_ an instance of the return type. But what if the return type is a protocol? 

_\*awkward silence\*_

Don't worry, we can fix this too. We cannot create an instance out of a protocol but we can make up a class that conforms to the protocol and return an instance of that. We could as well go ahead and generate these throwaway classes with Sourcery but I have got a better idea: let's assume that the protocols we are mocking may only return protocols which, too, have been mocked. If our mocks adhere to the same naming convention, we can choose to return the respective mock for a protocol return type, like so:

```swift
// in dummyValue(for:typeName:)
...
    else if let type = type, type.kind == "protocol", 
            type.annotations["mockable"] != nil {
        return mockClassName(for: type) + "() as " + typeName.name
    }
...
```

##### Going generic

For quite some time, I was under the impression that the template was complete until recently when I made an unconvenient discovery: Sourcery only has type information of the types it finds in the files we explicitly specify. That means that, apart from the name, it doesn't know anything about types coming from other modules (system frameworks included). This is bad news for us since the template won't recognize e.g. a `(NS)FileManagerDelegate` type as a protocol and will go on emitting a `.dummyValue` for it which, as we have seen, is impossible to satisfy. We need some mechanism that works for both protocols and instantiable types. After some experimentation I came up with this:

```swift
class Dummy<T> {}
```

The `Dummy` class does nothing but the `T` type parameter allows us to create specialized extensions depending on the type: 

```swift
// to create a dummy for MyProtocol:
class DummyForMyProtocol: MyProtocol { ... }

extension Dummy where T == MyProtocol {
    static var value: T { return DummyForMyProtocol() }
}

// to create a dummy for MyStruct:
extension Dummy where T == MyStruct {
    static var value: T { 
        return MyStruct(some: "meaningful", default: .values) 
    }
}
```

To refer to the dummy value, we can simply use `Dummy<MyProtocol>.value`. The good news is that we can make this semi-automatic for types with a parameterless `init()`:

```swift
protocol DefaultInitializable {
    init()
}

extension Dummy where T : DefaultInitializable {
    static var value: T { return T() }
}

extension String: DefaultInitializable {}
extension Int: DefaultInitializable {}
...
```

Let's go back to the `dummyValue(for:typeName:)` function and replace the `.dummyValue` case:

```swift
// in dummyValue(for:typeName:)
...
    else {
        return "Dummy<" + typeName.name + ">.value"
    }
}
```

And with that, we are really done (well, for now). The complete template can be downloaded [here](https://github.com/lvsti/MockSix/blob/master/Support/Mockable.swifttemplate). (There are some subtle differences though that are responsible for pretty printing the result.) I'm sure it could be further improved but I hope it already proves the point.

### Working with generated mocks

With the above infrastructure in place, mocking a new protocol consists of the following steps:

1. add the `// sourcery: "mockable"` annotation to the protocol for the template to pick the type up
2. run Sourcery and update the generated mocks file with the output
3. check if there are any new types that need to be extended for `Dummy<T>` (the compiler will tell you), and if so, provide a `static var value` for them (or, alternatively, conform them to `DefaultInitializable`)

That's all! The best thing is that all this can be done in less than a minute in the ideal case (no #3), and in a matter of few minutes if some plumbing is needed. With a bit of tweaking, even the conformance to `DefaultInitializable` could be generated for the known types that offer a parameterless `init()` if, during mock generation, we take note of the types we emitted a `Dummy<T>.value` for (here denoted as `dummyValueTypes`):

```swift
for (name, (type, typeName)) in dummyValueTypes {
    guard let type = type else {
        // ignore unknown types
        continue
    }

    let suitableInits = type.initializers.filter { 
        $0.isInitializer && 
        !$0.isFailableInitializer && 
        !$0.throws && 
        !$0.rethrows && 
        $0.parameters.isEmpty
    }

    guard !suitableInits.isEmpty else {
        continue
    }

    result += "extension " + name + ": DefaultInitializable {}"
}
```

### Conclusion

MockSix is a nice little framework which has proven being worthy on its own, but it can really shine combined with the boilerplate generation capabilities of Sourcery. What I most like in this setup is that Sourcery, while indeed does a terrific job, hasn't become yet another dependency for our projects. We could decide to abandon it anytime and still continue to write mocks the same way as before (this isn't the case e.g. with [Cuckoo](https://github.com/SwiftKit/Cuckoo) where code generation and expectation testing are tightly coupled).

Unit testing in Swift is still not an easy ride and we are only at the beginning of the road, but with these tools in our hands that road definitely looks less bumpy.
