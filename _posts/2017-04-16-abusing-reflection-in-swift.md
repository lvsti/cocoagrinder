---
layout: post
disqus: true
title:  "Abusing reflection in Swift"
date:   2017-04-16 17:19:00 +0200
excerpt: "Even though Swift's reflection API is somewhat limited, it can still help reduce the boilerplate in some cases. Kids, don't try this at home."
---

The other day I watched Michael Helmbrecht's [Foolproof notifications talk](https://realm.io/news/michael-helmbrecht-slug-no-more-typos-foolproof-notifications-swift) and I also played a bit with the [playground project](https://github.com/mrh-is/PugNotification). I really liked the trick about the [curried enum constructors](https://github.com/mrh-is/PugNotification/blob/master/PugKitPlayground.playground/Sources/TypedNotification.swift#L34-L39), however, I found the presented way of conforming to the `TypedNotification` protocol [more than tedious](https://github.com/mrh-is/PugNotification/blob/master/PugKitPlayground.playground/Sources/PugNotification.swift#L20-L50). So I set off to see whether there is anything I could improve there.

### Getting the notification's name

For some time now, the Swift compiler is able to generate the `rawValue` for enums with string `RawValue` type, but that only works if the cases don't have any associated value. So this is okay:

```swift
enum FridgeNotification: String {
    case temperatureChanged
    case coolingStarted
    case coolingStopped
    case outOfMilk
}

let name = FridgeNotification.outOfMilk.rawValue // "outOfMilk"
```

But this won't compile:

```swift
enum FridgeNotification: String {
    case temperatureChanged(value: Int)
    case coolingStarted
    case coolingStopped(afterDuration: TimeInterval, consumption: Double)
    case outOfMilk
}

// ERROR: 'FridgeNotification' declares raw type 'String', 
//   but does not conform to RawRepresentable and 
//   conformance could not be synthesized
```

In his talk, Michael suggested conforming the notification enums to a custom `TypedNotification` protocol that requires them to assign a name to each case, like so:

```swift
protocol TypedNotification {
    var name: String { get }
    ...
}

extension FridgeNotification: TypedNotification {
    var name: String {
        switch self {
        case .temperatureChanged: return "Fridge.temperatureChanged"
        case .coolingStarted: return "Fridge.coolingStarted"
        case .coolingStopped: return "Fridge.coolingStopped"
        case .outOfMilk: return "Fridge.outOfMilk"
        }
    }
}
```

Even though we only have to type the strings once, this solution does not scale well. So how could we get a unique identifier for each enum case *programmatically*?

Swift has very limited reflection abilities but it turns out that they suffice for this very task. Let's look at what we get when we create a mirror for an enum case:

```swift
let notif = FridgeNotification.temperatureChanged(value: 5)

let mirror = Mirror(reflecting: notif)

print("\(mirror.children)")         
// -> AnyCollection<(label: Optional<String>, value: Any)>(...)

print("\(mirror.children.first!)")  
// -> (label: Optional("temperatureChanged"), value: 5)

print("\(mirror.children.first!.label!)")  
// -> "temperatureChanged"
```

Neat! Not only we are provided with the name of the actual enum case, we also get access to the associated value(s). (This will come handy later.) Thus, nothing can stop us from writing the following code:

```swift
extension TypedNotification {
    private static func stringify(_ notif: Self) -> String {
        let m = Mirror(reflecting: notif)
        let caseName = m.children.isEmpty ? "\(notification)" : m.children.first!.label!
        let name = "\(type(of: notification))_\(caseName)"
        return name
    }

    static func post(_ notif: Self) {
        let name = Notification.Name(rawValue: stringify(notif))
        ...
    }
}
```

There is an extra check for bare enum cases that, curiously enough, don't expose any logical children. It seems that the rule is that all mirror children must have a value, and bare enums obviously don't satisfy that requirement. I could imagine `children` to be of type `AnyCollection<(label: String?, value: Any?)>` which would cover this case as well... but anyway. With that in place, we have just eliminated the need for the `name` property in `TypedNotification`. 🎉

### Dealing with the notification payload

Notifications frequently carry extra information in their `userInfo` property. Michael's clever trick allows for both passing and extracting the notification content in a strongly typed manner. However, the presented solution has an unfortunate limitation: only one associated value is supported. This would force us to define payload structs to use with notifications that convey multiple values (tuples are ruled out since they are not default initializable). Moreover, conforming types are required to implement a `content` property that establishes the binding between the enum case's associated value and the notification payload, which, again, is a lot of boilerplate:

```swift
protocol TypedNotificationContentType {
    init()
}
protocol TypedNotification {
    var content: TypedNotificationContentType? { get }
    ...
}

struct FridgeCoolingStoppedPayload {
    let duration: TimeInterval = 0.0
    let consumption: Double = 0.0
}

extension FridgeCoolingStoppedPayload: TypedNotificationContentType {}

extension FridgeNotification: TypedNotification {
    var content: TypedNotificationContentType? {
        switch self {
        case .temperatureChanged(let temp): return temp
        case .coolingStopped(let duration, let consumption): 
            return FridgeCoolingStoppedPayload(duration: duration,
                                               consumption: consumption)
        default: return nil
        }
    }
}
```

After some unfruitful trials, I managed to further abuse the reflection API to circumvent this limitation, at the same time rendering the `content` property unnecessary.

Let's start out with the unary case which is where the magic happens.

```swift
extension TypedNotification {
    ...
    static func addObserver<T1>(_ notif: (T1) -> Self,
                                using block: @escaping (T1) -> Void) -> ObserverToken
        where T1 : TypedNotificationContentType {
        let name = Notification.Name(rawValue: stringify(notif(T1())))
        let observer = NotificationCenter.default.addObserver(forName: name, object: nil, queue: nil) { notif in
            if let arg1 = notif.userInfo?["arg1"] as? T1 {
                block(arg1)
            }
        }
        return ObserverToken(observer: observer)
    }
}
```

So far nothing new compared to Michael's solution. Let's try to make it binary just by cloning the arguments:

```swift
extension TypedNotification {
    ...
    static func addObserver<T1, T2>(_ notif: (T1, T2) -> Self,
                                    using block: @escaping (T1, T2) -> Void) -> ObserverToken
        where T1 : TypedNotificationContentType, 
              T2 : TypedNotificationContentType {
        
        let name = Notification.Name(rawValue: stringify(notif(T1(), T2())))
        let observer = NotificationCenter.default.addObserver(forName: name, object: nil, queue: nil) { notif in
            if let arg1 = notif.userInfo?["arg1"] as? T1,
               let arg2 = notif.userInfo?["arg2"] as? T2 {
                block(arg1, arg2)
            }
        }
        return ObserverToken(observer: observer)
    }
}
```

This could actually work but how do we arrange the associated values into the `arg1`, `arg2`, ..., `argN` fields of the userInfo dictionary? In the `post(_:)` method, we are given a polymorphic enum value that we could as well `switch` on but then we are back to square one. Let's try something else instead. Remember just a while ago when we saw that mirrors knew about associated values as well? Pepperidge farm remembers. However, it's not all rainbows and unicorns:

```swift
let notif0 = FridgeNotification.outOfMilk
let mirror0 = Mirror(reflecting: notif0)
print("\(mirror0.children.first)")
// -> nil

let notif1 = FridgeNotification.temperatureChanged(value: 5)
let mirror1 = Mirror(reflecting: notif1)
print("\(mirror1.children.first!.value)")
// -> 5

let notif2 = FridgeNotification.coolingStopped(afterDuration: 10.0, 
                                               consumption: 42.0)
let mirror2 = Mirror(reflecting: notif2)
print("\(mirror2.children.first!.value)")  
// -> (afterDuration: 10.0, consumption: 42.0)
```

Depending on the arity of the enum case, we can get back a tuple, a scalar, or no value at all, so we have to be careful not to crash the app with an imprudent forced unwrap. The following code builds up the `userInfo` dictionary with that in mind:

```swift
var userInfo: [String: Any]? = nil

let notifMirror = Mirror(reflecting: notif)
if !notifMirror.children.isEmpty {
    // has associated value    
    let value = notifMirror.children.first!.value
    
    // check what we've got
    let valueMirror = Mirror(reflecting: value)

    if valueMirror.children.isEmpty {
        // unary case, value is scalar
        userInfo = ["arg1": value]
    } else {
        // n-ary case, walk the tuple
        userInfo = [:]
        for (i, item) in valueMirror.children.enumerated() {
            userInfo?["arg\(i + 1)"] = item.value
        }
    }
}
```

The result is a dictionary of the enum's associated values nicely arranged, exactly the way that `addObserver` expects them. Now we can dump the `content` property requirement as the above code takes care of the payload, and also lets us use multiparameter closures on the observer side:

```swift
_ = FridgeNotification.addObserver(FridgeNotification.coolingStopped) { duration, consumption in
    // `duration` is inferred to be TimeInterval, and `consumption` a Double
    ...
}
```

Note that due to the static nature of the type-safe extraction, we'll have to define an overload for `addObserver` for each arity that we plan to support. That is, if we want to have enums with 4 associated values, there has to be an `addObserver<T1,T2,T3,T4>` implementation that handles that case.

### Final thoughts

We've seen that it is possible to leverage Swift's reflection API to get rid of some of the boilerplate. Note, however, that this hack is extremely fragile and is probably not suitable for production code. As of now, reflection is meant to be used solely for debugging and there are no guarantees that an upcoming Swift update doesn't silently change the string representation or the mirror structure of the reflected types, which would eventually render the above solution invalid.

The tweaked version of Michael's playground is available [here](https://github.com/lvsti/PugNotification/tree/reflection_code_golf).
