---
title: Archiving and Unarchiving Swift Structure Instances — Revisited
date: "2015-12-08T22:12:03.284Z"
description: "A post illustrating how to archive and unarchive structures in Swift."
---

![A cover image showing a grid of archival drawers](images/1_9Ss8_0jGEJjNk5gXMFdEtQ.jpeg)

> Note — Since the release of Swift 3, a few statements made in this article are not true for Swift 3, but they hold good for older versions of Swift.

The power of structures in Swift makes you want to use it a lot more than you would in any other language. This is a good thing, as long as you know when to use structures and when to strictly not.

One disadvantage of structures is that it cannot conform to NSCoding protocol, which requires its conformers to be of classes. This means, there’s not straight-forward way to archive and unarchive classes.

When working on a project, I needed to archive structure instances. I wrote another article about how I ended up doing it. I have been doing more research ever since and now have discovered three ways of doing it, and I like the third way the best.

### Approach 1

Let’s say I have a structure Movie,
```swift
struct Movie {

    let name: String
    let director: String
    let releaseYear: Int

}
```

Create a class mimicking the structure and use the class to archive and unarchive. Unarchive the class and create an instance of the structure using the class object.

```swift
class MovieClass: NSObject, NSCoding {

    var movie: Movie?

    required convenience init?(coder aDecoder: NSCoder) {
        self.init()
        if let name = aDecoder.decodeObjectForKey(“name”) as? String,
            director = aDecoder.decodeObjectForKey(“director”) as? String,
            releaseYear = aDecoder.decodeObjectForKey(“releaseYear”) as? Int {
            movie = Movie(name: name, director: director, releaseYear: releaseYear)
        }
    }

    func encodeWithCoder(aCoder: NSCoder) {
        if let movie = movie {
            aCoder.encodeObject(movie.name, forKey: “name”)
            aCoder.encodeObject(movie.director, forKey: “director”)
            aCoder.encodeObject(movie.releaseYear, forKey: “releaseYear”)
        }
    }

}
```

As seen above, the class `MovieClass` conforms to `NSCoding` protocol. Hence, the `encodeWithCoder:` and a failable initializer are defined as methods in the class. The encodeWithCoder: method assigns a key for each property of the struct movie and encodes them as individual objects. The initializer safely unwraps all three properties of the class, and only if all three are not `nil`, it assigns its movie property with a newly created structure instance of Movie with the obtained values.

We then have two external methods, one to archive and the other to unarchive the structure instance. Refer to the playground [ArchivingSwiftStructures1](https://github.com/vishalvshekkar/ArchivingSwiftStructures) for the complete code.

```swift
func archiveMovie(movie: Movie) {
    let movieClassObject = MovieClass()
    movieClassObject.movie = movie
    NSKeyedArchiver.archiveRootObject(movieClassObject, toFile: /* Some file path */)
}

func unarchiveMovie() -> Movie? {
    let movieClassObject = NSKeyedUnarchiver.unarchiveObjectWithFile(/* Some file path */) as? MovieClass
    return movieClassObject?.movie
}
```

This gets the work done, it’s not elegant, and can only be considered as a dirty fix because you’ve a class and a structure defined for the same kind of data to be archived and unarchived.

### Approach 2

This way is not a very big improvement from the last, but, it avoids exposing the class `MovieClass` outside of the structure. Here, the class is utilized for the sole purpose of archiving and unarchiving the structure instance. This is ensured by defining the class in an extension of the structure. By doing this, only the structure instances gain access to the class, and hence avoiding inappropriate archiving with a similar key by any other method.
First, let’s write two more methods in our struct, Movie. Let’s have an archive and a static unarchive method.

```swift
struct Movie {

    let name: String
    let director: String
    let releaseYear: Int

    func archive() {
        let movieClassObject = MovieClass(movie: self)
        NSKeyedArchiver.archiveRootObject(movieClassObject, toFile: /* Some file path */)
    }

    static func unarchive() -> Movie? {
        let movieClassObject = NSKeyedUnarchiver.unarchiveObjectWithFile(/* Some file path */) as? MovieClass
        return movieClassObject?.movie
    }
    
}
```

Next, we extend the struct `Movie` and define a class `MovieClass` inside the extension.

```swift
extension Movie {

    class MovieClass: NSObject, NSCoding {

        var movie: Movie?
        
        init(movie: Movie) {
            self.movie = movie
            super.init()
        }

        required init?(coder aDecoder: NSCoder) {
            ...
        }

        func encodeWithCoder(aCoder: NSCoder) {
            ...
        }

    }

}
```

We can also make the `MovieClass` private, but, this requires the failable initializer and `encodeWithCoder:` to be tagged with `@objc`.

Refer to the playground [ArchivingSwiftStructures2](https://github.com/vishalvshekkar/ArchivingSwiftStructures) for the complete code.

### Approach 3

What we do here is define a protocol `Dictionariable` which contains two non-optional methods to be implemented, `dictionaryRepresentation` and a failable initializer.

```swift
protocol Dictionariable {

    func dictionaryRepresentation() -> NSDictionary
    init?(dictionaryRepresentation: NSDictionary?)
    
}
```

We use `NSDictionary`, instead of `Dictionary`, as the former does not support objects of type `Any`. This is required because `NSKeyedUnarchiver` can only perform its functions on `AnyObject` type and not on `Any` type. By using `NSDictionary`, we ensure we get a compiler error if an object which does not conform to `AnyObject` protocol is used instead of getting a silent failure during unarchiving and finding the object to be `nil`.

Now, let’s extend the structure `Movie` and conform it to `Dictionariable` protocol.

```swift
extension Movie: Dictionariable {

    func dictionaryRepresentation() -> NSDictionary {
        let representation: [String: AnyObject] = [
        “name”: name,
        “director”: director,
        “releaseYear”: releaseYear
        ]
        return representation
    }

    init?(dictionaryRepresentation: NSDictionary?) {
        guard let values = dictionaryRepresentation else {return nil}
        if let name = values[“name”] as? String,
            director = values[“director”] as? String,
            releaseYear = values[“releaseYear”] as? Int {
            self.name = name
            self.director = director
            self.releaseYear = releaseYear
        } else {
            return nil
        }
    }
    
}
```

By implementing the above two methods, we’ve ensured that each property of the structure get’s converted into an `NSDictionary` which the `NSKeyedUnarchiver` loves. We also have a failable initializer for the structure that initializes it from the unarchived `NSDictonary`.

The beauty of this method comes in using generics to archive and unarchive `Dictionariable`-conforming structures. By using generics, we are able to archive and unarchive all structures conforming to `Dictionariable`. All we need to do is conform every structure that needs archiving to `Dictionariable` protocol and the following two methods will handle the rest.

```swift
func extractStructureFromArchive<T: Dictionariable>() -> T? {
    guard let encodedDict = NSKeyedUnarchiver.unarchiveObjectWithFile(/* Some file path */) as? NSDictionary else {return nil}
    return T(dictionaryRepresentation: encodedDict)
}

func archiveStructure<T: Dictionariable>(structure: T) {
    let encodedValue = structure.dictionaryRepresentation()
    NSKeyedArchiver.archiveRootObject(encodedValue, toFile: /* Some file path */)
}
```

The `extractStructureFromArchive` method will unarchive and return an optional instance of the `Dictionariable` conforming structure. The caveat is that the return type of the method is not implicitly known, so we have to explicitly typecast the variable holding the returned value, or do an ‘as’ operation.

```swift
let someMovie: Movie? = extractStructureFromArchive()
```

### Bonus
Because we use generics, we can use the same methods to archive and unarchive nested structures. It works just like `NSCoding` works, as long as all the structures are `Dictionariable`-conforming.

```swift
struct Actor {

    let name: String
    let firstMovie: Movie

}

extension Actor: Dictionariable {

    func dictionaryRepresentation() -> NSDictionary {
        let representation: [String: AnyObject] = [
        “name”: name,
        “firstMovie”: firstMovie.dictionaryRepresentation(),
        ]
        return representation
    }

    init?(dictionaryRepresentation: NSDictionary?) {
        guard let values = dictionaryRepresentation else {return nil}
        if let name = values[“name”] as? String,
            let someMovie: Movie = extractStructureFromDictionary() {
            self.name = name
            self.firstMovie = someMovie
        } else {
            return nil
        }
    }
    
}
```

In the example above, the structure `Actor` has a property called `firstMovie` which is of type `Movie` structure. The above conformance of `Dictionariable` protocol by the `Actor` structure enables it to be archived and unarchived just like the struct `Movie` before.
Refer to the playground [ArchivingSwiftStructures3](https://github.com/vishalvshekkar/ArchivingSwiftStructures) for the complete code.

![A minions meme about tiredness](images/1_LaVZdljPX5BuLSPpm0Iu4Q.jpeg)

Swift introduced a tremendous amount of type-safety. But, archiving and unarchiving always loses the types of the objects. Until a better way comes along to do this by supporting all of Swift’s principles, we should make do with what we have.

Go through the relevant playground files over here https://github.com/vishalvshekkar/ArchivingSwiftStructures
They contain a feature-rich implementation of the same with some code showing results.

And, also check out my other projects at https://github.com/vishalvshekkar
Ciao.