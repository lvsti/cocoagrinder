---
layout: post
title:  "Realm – Use Now, Commit Yourself Later"
date:   2016-05-07 14:12:35 +0200
categories: ios swift
excerpt: "Avoiding technological lock-in while testdriving the Realm database aka. how to have your cake and eat it too?"
---

What do you do when you need to store and retrieve large amounts of data in an iOS app? Your first thought would probably be to use CoreData as it comes directly from Apple and, thus, is deeply integrated with the ecosystem. The Cocoa Way (TM) of handling data, so to speak. I have burnt myself with CoreData various times in my developer career but still, shortly after kicking off my current iOS [pet project](https://github.com/lvsti/Towel), the `.xcdatamodel` was already bustling with models and entity relations when I first stopped to think: what if this time I did it another way?

I had no previous experience with the [Realm](https://realm.io) mobile database but I'm a frequent visitor of their Swift video collection, and was curious to find out if their solution is less of a P.I.T.A. than CoreData is, so I decided to give it a try. 

Taking a quick look at the documentation and checking out the samples revealed to me that Realm's API deviates significantly from other Cocoa frameworks. Take the query results for example: while `NSManagedObjectContext.executeFetchRequest(_:)` returns an ordinary array, Realm's `objects(_:)` gives back `Results<T>`, an esoteric collection type that's not even that (it doesn't conform to `CollectionType` or any of the Swift protocols). But just by using the resulting model objects you can easily qualify for a framework lock-in:

```swift
import RealmSwift

class DBPlaceInfo: Object {
    dynamic var _placeID: Int32 = 0
    let _altitude = RealmOptional<Double>()
    
    dynamic var _location: DBLocation?
    let _comments = List<DBPlaceComment>()

    let _place = LinkingObjects(fromType: DBPlace.self, property: "_placeInfo")
    }
}
```

Long story short, switching to Realm as-is would have been a leap of faith: if later it had turned out that it didn't suit my needs, I would have had to rewrite all the code that made use of the queries or their results.

So how did I proceed? My goal was to hide any implementation detail of the persistence layer (model objects included) so that the frontend didn't have to know how the data is fetched or arranged. On the other hand, I wanted to avoid the performance hit that would have arisen if I had just copied the data at the persistence layer&ndash;app logic boundary. So I turned to protocols for help.

My idea was to create a protocol for each model object and pass around those instead of the actual classes/structs. For the above model, I came up with this:

```swift
protocol PlaceInfo {
    var placeID: Int32 { get }
    var altitude: Double? { get }
    
    var location: Location? { get }
    var comments: [PlaceComment] { get }
    
    var place: Place { get }
}
```

Cool, no traces of Realm-specific types here (`Location`, `PlaceComment`, and `Place` are protocols similar to `PlaceInfo`). Now I could make `DBPlaceInfo` conform to this wrapper protocol:

```swift
extension DBPlaceInfo: PlaceInfo {
    var placeID: Int32 { return _placeID }
    var altitude: Double? { return _altitude.value }
    
    var location: Location? { return _location }
    
    var comments: [PlaceComment] {
        // ummm... now what?
    }

    var place: Place { return _place.first! }
}
```

Wiring properties and one-to-one relationships was a cakewalk; but how could I map the one-to-many `comments` from Realm's `List<T>` (another esoteric container) to an iterable and O(1)-countable collection (like an array)?

In my first trial, I squeezed an `AnyGenerator<U>` out of `List<T>` as follows:

```swift
    var comments: AnyGenerator<PlaceComment> {
        let gen = _comments.generate()
        return AnyGenerator { return gen.next() as? PlaceComment }
    }
```

This is now iterable but the only way to get the count is to walk through the generated sequence. I guess it's needless to say that this was not a viable solution for result sets in the ballpark of 10,000 rows. I could have used this workaround:

```swift
    var comments: (AnyGenerator<PlaceComment>, Int) {
        let gen = _comments.generate()
        return (AnyGenerator { return gen.next() as? PlaceComment }, _comments.count)
    }
```

but, apart from it being terribly ugly, it's still nowhere to support random access. Imagine scrolling through a `UITableView` where each `tableView(_:cellForRowAtIndexPath:)` had to rewind and fast-forward  to the row in question... There has to be a better way.

By this time, I already knew that what I wanted was something conforming to `CollectionType`:

```swift
struct ToMany<U>: CollectionType {
    // access a List<T> through a ToMany<U>, where T : U
    // but how?
}
```

I spent a couple of hours juggling with `AnyRealmCollection`, `AnyCollectionType`, and `AnyRandomAccessCollection`, but either the types did not match or the resulting artifact was leaking information of the underlying implementation, which is what I wanted to avoid in the first place.

The solution hit me in the shower: instead of passing in the `List<T>`, which would inevitably leave its marks on the type of `ToMany`, I just need to give `ToMany` access to two things: `List<T>.count` and `List<T>.subscript(_:)`. Like how? With closures!

```swift
struct ToMany<U>: CollectionType {
    private let _countFunc: () -> Int
    private let _subscriptFunc: (Int) -> U?
    
    init(countFunc: () -> Int, subscriptFunc: (Int) -> U?) {
        _countFunc = countFunc
        _subscriptFunc = subscriptFunc
    }
    
// from Indexable:
    
    typealias Index = Int
    
    var startIndex: Int {
        return 0
    }
    
    var endIndex: Int {
        return _countFunc()
    }
    
// from SequenceType:
    
    typealias _Element = U
    
// from CollectionType:
    
    subscript (position: Int) -> U {
        return _subscriptFunc(position)!
    }
    
    var isEmpty: Bool { return count == 0 }
    var count: Int { return endIndex }
    var first: U? { return _subscriptFunc(0) }
}
```

With this in my toolbox, I was now able to map the Realm model object collection to a purely protocol-based one with no significant overhead:

```swift
    var comments: ToMany<PlaceComment> {
        return ToMany<PlaceComment>(
            countFunc: { return self._comments.count },
            subscriptFunc: { self._comments[$0] }
        )
    }
```

Surprisingly enough, if one day I had to go back to CoreData and get the `_comments` one-to-many relationship as an `NSOrderedSet`, the above code would still hold without having to change a single character, thanks to `count` and `subscript` being available on that class as well.

At this point, model objects and relationships had been taken care of, the only thing remaining was to wrap the actual queries. For that, I created a `Query` class that listed all possible queries that could be made:

```swift
class Query {
    // [boilerplate omitted for clarity]
    
    static func getAllPlaces() -> ToMany<Place> {
        let results = realm().objects(DBPlace)
        return ToMany<Place>(
            countFunc: { return results.count }, 
            subscriptFunc: { results[$0] }
        )
    }
}
```

I could push this even further and define a protocol that completely hides the innards of the `Query` class. Either way, provided the app logic communicates with the data layer via this interface, you could swap implementations in and out without having to change a single line of code elsewhere.

Are we happy, Vincent? Sure we are.
