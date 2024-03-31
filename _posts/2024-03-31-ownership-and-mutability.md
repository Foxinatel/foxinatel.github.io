---
layout: post
title:  "Considerations on Ownership and Mutability in Golang"
date:   31/03/2024
---

As part of my second year of university I had to learn some Golang. To be completely honest... I hated it, there were a few features about the language that just never sat right with me. One of those things was the lack of separation between owned and non-owned memory.

One of the first things that my university warned us about when teaching the language was about arrays. Let's take a quick look at the following:
```go
arr := []int{1,2,3}
fmt.Println(arr)
subarr := arr[:2]
subarr[1] += 1
fmt.Println(arr)
```

The output of the first `fmt.Println` is obviously `[1 2 3]`. To someone who doesn't know Go, there's two reasonable answers for the second line out:
- `[1 2 3]`: The subarr reallocates and therefore mutating it has no impact on the original
- `[1 3 3]`: The subarr references the existing array and therefore mutating it mutates the original

In the case of Golang, it chooses the second option. Personally I'd consider this to be a good move overall because it avoids reallocations of potentially massive arrays.

Now in my brief time with Golang there was another thing that I came to learn quite quickly, which is that there isn't really a distinction in the type-system between owning and non-owning arrays. To those who aren't really aware what I mean by this, let's take a quick detour through some rust code that does the same as what we saw above.

```rs
let mut arr = vec![1,2,3];
println!("{arr:?}");
let subarr = &mut arr[..2];
subarr[1] += 1;
println!("{arr:?}");
```

Now, if we leave the rust compiler to figure out the types for `arr` and `subarr` for us, it tells us that `arr` is of type `Vec<i32>`, while `subarr` is of type `&mut [i32]`. This can lead us onto a long talk about ownership, but for the sake of brevity, all that you really need to know here is that a `Vec<T>` "owns" its array, meaning it can do operations like appending new elements onto itself. Conversely, a `&mut [T]` *doesn't* have ownership, meaning we're limited to only updating values within the fixed bounds of our slice. We can't resize our slice or perform any mutation outside its bounds.

However if we look back to our Go code... we don't have any sort of distinction between the types for `arr` and `subarr`, they are both `[]int`. So knowing this left me with one big question: what does it mean to perform resizing operations on a slice of another bit of memory?

### Clobbering Memory Outside Sub-Slices

To explain this, let's look at some code again.

```go
arr := []int{1,2,3}
fmt.Println(arr) // [1 2 3]
arr = append(arr, 4, 5)
fmt.Println(arr) // [1 2 3 4 5]
subarr := arr[:3]
subarr = append(subarr, 6, 7)
fmt.Println(arr) // ???
```

Now my immediate first guess here would be that as soon as you try to extend a sub-slice, it would reallocate and move itself onto the heap. This seems like the only sensible option, as it would prevent unexpectedly modifying values that aren't inside our slice.

Surely any sane language wouldn't start clobbering it's own memory... Surely...

... And yet the output is [1 2 3 6 7]

... Oh no

This feels like it has some serious potential for footgunning. I've been playing around with a voice recognition library recently so lets create a small contrived example in light of that:
```go
func create_recognizer(grammar []string) Recognizer {
    // Given any grammar, we want to have a cancel option
    // We append "cancel" to our grammar
    grammar = append(grammar, "cancel")
    // Setup and return a voice recognizer
}

func main() {
    arr := []string{"foo", "bar", "baz", "qux"}
    // Create a model that just recognizes "foo" or "bar"
    recognizer_1 := create_recognizer(arr[:2])
    // Create a model that just recognizes "baz" or "qux"
    recognizer_2 := create_recognizer(arr[2:4])
}
```

Now this code seems pretty harmless overall, but within it is a fairly nasty bug. After we call `create_recognizer(arr[:2])`, the value of `arr` has actually been modified... outside of the bounds of the slice we gave it. If we call `fmt.Println(arr)` between the two calls to `create_recognizer`, we get `[foo bar cancel qux]`. So when we call `create_recognizer(arr[2:4])`, we're actually passing in the array `[cancel qux]`. This means that `baz` will *never* be added to the grammar of our voice recognizer. Oh dear...

I can already feel my feet aching from the all the potential bullets that could hit it right now.

But with all this I had a new question. What happens if we append to a slice beyond the bounds of the parent?

### Silent Ownership Promotions

Let's make a new example that takes some inspiration from our first bit of code, but this time with a loop and some appending:

```go
arr := []int{1,2,3,4,5}
fmt.Println(arr) // [1 2 3 4 5]
subarr := arr[:2]

subarr[1] += 1
fmt.Println(arr) // [1 3 3 4 5]

for i := 0; i < 2; i++ {
    subarr = append(subarr, 6, 7)
    fmt.Println(arr)
    // First iteration: [1 3 6 7 5]
    // Second iteration: ???

    subarr[1] += 1
    fmt.Println(arr)
    // First iteration: [1 4 6 7 5]
    // Second iteration: ???
}
```

Now the answer to this one is a bit more obvious, but I still feel like it's a massive oversight in the design of the language. On the second loop, appending a new `6,7` to `subarr` extends it beyond the bounds of `arr`, meaning that it only really has one option; allocate on the heap and copy the array across. Of course this now silently promotes `subarr` from being a non-owning array to being an owning array. Because it's no longer a non-owning array, making modifications to `subarr` now *doesn't* update `arr`. This means that on the second iteration of the loop, both `fmt.Println(arr)` print out `[1 4 6 7 5]`.

Putting this all together, we have a value `arr[1]` that enters a loop with a value of `3`, the loop happens twice, and inside that loop is the statement `subarr[1] += 1`. At the end of the loop we have `arr[1]` with a value of `4`. Ew.

### A Fun but Dangerous Trick

Now, a crafty programmer that really hates reallocations might come along and try to exploit this behaviour. Let's imagine some contrived interpolation function:

```go
func interpolate(arr []float32) []float32 {
    // This contrived example is a really simple linear interpolation
    // But realistically this could be some giant polynomial solver that uses dozens of points
    diff := arr[1] - arr[0]
    end := len(arr) - 1
    return append(arr, arr[end]+1*diff, arr[end]+2*diff, arr[end]+3*diff)
}

func main() {
    arr := []float32{1, 5, 9}
    fmt.Println(arr)
    fmt.Println(interpolate(arr))
}
```

This quite reasonably prints out
```
[1 5 9]
[1 5 9 13 17 21]
```

However we know that our `interpolate` function appends... and that could slow down our code due to memory allocations. So let's assume a crafty programmer comes along and exploits this "feature" of Go and does the following:

```go
func main() {
    arr := []float32{1, 5, 9, 0, 0, 0}
    subarr := arr[:3]
    fmt.Println(subarr)
    interpolate(subarr)
    fmt.Println(arr)
}
```

Well in this case we get back out
```
[1 5 9]
[1 5 9 13 17 21]
```

So everything is working, right! And we can also guarantee there's no accidental reallocations happening in `interpolate`!

...Well, I think it's a cool idea. Definitely the kinda thing I'd try to do given half a chance, but it's also terrifying. If anyone ever came along and changed the functionality of `interpolate` to return a different number of new points, suddenly the `append` would do one of the following:
- The append extends the subslice beyond the bounds of the parent array that we're trying to populate, and we end up with everything past our subslice uninitialized, as the append forced it to reallocate
- The append falls short of the array we're trying to populate, once again leaving us with uninitialized values at the end