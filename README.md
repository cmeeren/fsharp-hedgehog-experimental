# fsharp-hedgehog-experimental

[![NuGet][nuget-shield]][nuget] [![AppVeyor][appveyor-shield]][appveyor]

[Hedgehog][hedgehog] with batteries included: Auto-generators, extra combinators, and more.

<img src="https://github.com/cmeeren/fsharp-hedgehog-experimental/raw/master/img/SQUARE_hedgehog_615x615.png" width="307" align="right"/>

## Features

- Auto-generation of arbitrary types
- Generation of functions
- Lots of convenient combinators

## Examples

### Convenience combinators

**Generate lists without having to explicitly use `Range`:**

```f#
let! exponentialList = Gen.bool |> GenX.eList 1 5 // Same as Gen.list (Range.exponential 1 5)
let! linearList      = Gen.bool |> GenX.lList 1 5 // Same as Gen.list (Range.linear 1 5)
let! constantList    = Gen.bool |> GenX.cList 1 5 // Same as Gen.list (Range.constant 1 5)
```

**Generate strings without having to explicitly use `Range`:**

```f#
let! exponentialStr = Gen.alpha |> GenX.eString 1 5 // Same as Gen.string (Range.exponential 1 5)
let! linearStr      = Gen.alpha |> GenX.lString 1 5 // Same as Gen.string (Range.linear 1 5)
let! constantStr    = Gen.alpha |> GenX.cString 1 5 // Same as Gen.string (Range.constant 1 5)
```

**Generate a random shuffle/permutation of a list:**

```f#
let lst = [1; 2; 3; 4; 5]
let! shuffled = GenX.shuffle lst // e.g. [2; 1; 5; 3; 4]
```

(Note that the shuffle may produce an identical list; this is more likely for shorter lists.)

**Shuffle/permute the case of a string:**

```f#	
let str = "abcde"
let! shuffled = GenX.shuffleCase str // e.g. "aBCdE"
```

**Generate an element that is not equal to another element (direct or option-wrapped):**

```f#
// Generates an int that is not 0
let! notZero = Gen.int (Range.exponentialBounded()) |> GenX.notEqualTo 0

// Also generates an int that is not 0
let! notZero = Gen.int (Range.exponentialBounded()) |> GenX.notEqualToOpt (Some 0)

// Can generate any int
let! anyInt = Gen.int (Range.exponentialBounded()) |> GenX.notEqualToOpt None
```

**Generate a string that does not equal another string, start with another string, or is a substring of another string:**

```f#
let strGen = Gen.alpha |> GenX.lString 1 5

let! ex1 = strGen |> GenX.notEqualTo "A" // Does not generate "A" (same function as previous example)
let! ex2 = strGen |> GenX.iNotEqualTo "A" // Case insensitive, does not generate "A" or "a"
let! ex3 = strGen |> GenX.notSubstringOf "fooBar" // Does not generate e.g. "Bar" (but "bar" is OK)
let! ex4 = strGen |> GenX.iNotSubstringOf "fooBar" // Case insensitive, does not generate e.g. "Bar" or bar"
let! ex5 = strGen |> GenX.notStartsWith "foo" // Does not generate e.g. "foobar" (but "Foobar" is OK)
let! ex6 = strGen |> GenX.iNotStartsWith "foo" // Case insensitive, does not generate e.g. "foobar" or "Foobar"
```

**Generate an item that is not in a specified list:**

```f#
let! str = Gen.int (Range.exponentialBounded()) |> GenX.notIn [1; 2] // Does not generate 1 or 2
```

**Generate a list that does not contain a specified item:**

```f#
// Produces a list not containins 2, e.g. [1; 5; 7]. Note that this is a filter and not
// a removal after the list has been generated, so the length of the list is unaffected.
let! intList = Gen.int (Range.exponentialBounded()) |> GenX.cList 3 3 |> GenX.notContains 2
```

**Generate a list that contains a specified item at a random index:**

```f#
// Generates a list and inserts the element. The list is 1 element longer than it would otherwise have been.
let! intList = Gen.int (Range.exponentialBounded()) |> GenX.cList 3 3 |> GenX.addElement 2
```

**Generate null some of the time, or don't generate nulls:**

```f#
let strGen = Gen.alpha |> GenX.eString 1 5

// Generates null part of the time (same frequency as Gen.option generates None)
let nullStrGen = strGen |> GenX.withNull

// Does not generate null
let noNullStrGen = nullStrGen |> GenX.noNull


```

**Don't generate nulls:**

```f#
let nullGen = Gen.alpha |> GenX.eString 1 5 |> GenX.withNull  // Generates null some of the time
let noNullGen = nullGen
```

**Generate sorted/distinct tuples (2, 3 or 4 elements):**

```f#
// Can produce 'a', 'a', 'c' but not 'a', 'c', 'b'
let! x, y, z = Gen.alpha |> Gen.tuple3 |> GenX.sorted3

// Can produce 'b', 'a', 'c' but not 'b', 'b', 'c'
let! x, y, z = Gen.alpha |> Gen.tuple3 |> GenX.distinct3

// Strictly increasing - can produce 'a', 'b', 'c' but not 'a', 'a', 'c'
let! x, y, z = Gen.alpha |> Gen.tuple3 |> GenX.increasing3
```

**Generate a date range:**

```f#
// Generates two dates at least 1 and at most 10 days apart, each with random time of day.
// The interval increases linearly with the implicit size parameter.
let! d1, d2 = GenX.dateInterval (Range.linear 1 10)
```

**Generate a function:**

```f#
// Generates a list using inpGen together with a function that maps each of the distinct
// elements in the list to values generated by outGen. Distinct elements in the input list
// may map to the same output values. For example, [2; 3; 2] may map to ['A'; 'B'; 'A'] or
// ['A'; 'A'; 'A'], but never ['A'; 'B'; 'C']. The generated function throws if called with
// values not present in the input list.
let intGen = Gen.int (Range.exponentialBounded())
let charListGen = Gen.alpha |> GenX.eList 1 10
let! chars, f = charListGen |> GenX.withMapTo intGen
// chars : char list
// f : char -> int


// Generates a list using inpGen together with a function that maps each of the
// distinct elements in the list to values generated by outGen. Distinct elements
// in the input list are guaranteed to map to distinct output values. For example,
// [2; 3; 2] may map to ['A'; 'B'; 'A'], but never ['A'; 'A'; 'A'] or ['A'; 'B'; 'C'].
// Only use this if the output space is large enough that the required number of distinct
// output values are likely to be generated. The generated function throws if called with
// values not present in the input list.
let intGen = Gen.int (Range.exponentialBounded())
let charListGen = Gen.alpha |> GenX.eList 1 10
let! chars, f = charListGen |> GenX.withDistinctMapTo intGen
// chars : char list
// f : char -> int
```

### Auto-generation

**Generate any type automatically using default auto-generators for primitive types:**

```f#
// Can generate all F# types (unions, records, lists, etc.) as well as POCOs
// with mutable properties or constructors.

type Union =
  | Husband of int
  | Wife of string
  
type Record =
  {Sport: string
   Time: TimeSpan}
   
// Explicit type parameter may not be necessary if it can be inferred.
let! union = GenX.auto<Union>
let! record = GenX.auto<Record>

// Recursive types are supported. By default, recurses at most once (subject to change).
type Recursive =
  {OptChild: Recursive option
   LstChild: Recursive list}
let! recursive = GenX.auto<Recursive>
// E.g. {OptChild = Some {OptChild = None; LstChild = []}; LstChild = []}
// Note that you may need to adjust the defaults when any kind of sequence is involved
// (see below), since by default the range for generated sequences are Range.exponential 0 50
// (subject to change).
```

**Generate any type automatically and override default auto-generators:**

```f#
let! myVal = GenX.autoWith<MyType> {GenX.defaults with String = ...; SeqRange = ...}
```

The current defaults are (see Gen.fs for up-to-date list):

```f#
let defaults =
  { Byte = Gen.byte <| Range.exponentialBounded ()
    Int16 = Gen.int16 <| Range.exponentialBounded ()
    Int = Gen.int <| Range.exponentialBounded ()
    Int64 = Gen.int64 <| Range.exponentialBounded ()
    Double = Gen.double <| Range.exponentialBounded ()
    Decimal = Gen.double <| Range.exponentialBounded () |> Gen.map decimal
    Bool = Gen.bool
    Guid = Gen.guid
    Char = Gen.latin1
    String = Gen.string (Range.linear 0 50) Gen.latin1
    DateTime = Gen.dateTime
    DateTimeOffset = Gen.dateTime |> Gen.map System.DateTimeOffset
    SeqRange = Range.exponential 0 50
    RecursionDepth = 1 }
```

[hedgehog]: https://github.com/hedgehogqa/fsharp-hedgehog

[nuget]: https://www.nuget.org/packages/Hedgehog.Experimental/
[nuget-shield]: https://img.shields.io/nuget/dt/Hedgehog.Experimental.svg?style=flat

[appveyor]: https://ci.appveyor.com/project/cmeeren/fsharp-hedgehog-experimental/
[appveyor-shield]: https://ci.appveyor.com/api/projects/status/9j83svr5wu0ydr23/branch/master?svg=true
