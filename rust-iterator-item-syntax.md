# Rust Iterator Items <br/><small>An exploration of syntax</small>

*TL;DR: [I think we should add generators to Rust](#Proposal-Iterator-Items). I’ve implemented [a prototype of my proposal using a procedural macro][iterator_item], and I would love people to [open issues][file tickets] and/or [PRs with implementations or potential syntax][PRs] and other ideas around the syntax of the feature.*


One of the nicest APIs available in Rust (and many other languages) is the `Iterator` trait, the `std` library implementation of the [Iterator pattern]. It allows you to abstract yourself from low-level details of container types, making it so you never have to write `for(i=0; i++; i<len)` and indexing operations, instead going through the provided API one element at a time using the `Iterator::next(&mut self) -> Self::Item` method.

In Rust and other languages, this functionality is considered so important, that `for`-loops take advantage of knowing its API surface and desugar the an expression you could write, like `for i in foo { ... }`, to a lowered expression closer to `let mut foo = foo.iter(); while let Some(i) = foo.next() { ... }`. This means that you use `Iterator` even if you're not conscious of it, and that custom container types can be iterated with `for`-loop expressions in the "expected" way by `impl`ementing `Iterator` for them.

But there's one catch, that might be best illustrated with an example. Take the "[merge overlapping intervals]" interview question. We won't explain the problem in detail, beyond a brief description: *"given an iterable of intervals where `intervals[i] = Interval { start, end }`, merge all overlapping intervals and return them"*.

If you were to try and solve it in the naïve way using `Vec`, it would look something like the following:

```rust
fn merge_overlapping_intervals(input: Vec<Interval>) -> Vec<Interval> {
   let mut result = Vec::with_capacity(input.len());
   if input.is_empty() {
       return result;
   }
   let mut prev = input[0];
    for i in input {
        if prev.overlaps(&i) {
            prev = prev.merge(&i);
        } else {
            result.push(prev);
            prev = i;
        }
    }
    result.push(prev);
    result
}
```

The only problem with such a solution is that it assumes that the `input` is bounded in size, and that it fits entirely in memory. Using `Iterator`s, we can change this O(n) space algorithm, to something that is still O(n) in time, but O(1) in space. But there's one hurdle: to write an `Iterator` today you need to [implement all the logic yourself][Iterator impl]:

```rust
struct MergeOverlappingIntervals<I: Iterator<Item = Interval>> {
    input: I,
    prev: Option<Interval>,
}

impl<I: Iterator<Item = Interval>> MergeOverlappingIntervals<I> {
    fn new(input: I) -> Self {
        MergeOverlappingIntervals { input, prev: None }
    }
}

impl<I: Iterator<Item = Interval>> Iterator for MergeOverlappingIntervals<I> {
    type Item = Interval;

    fn next(&mut self) -> Option<Self::Item> {
        while let Some(i) = self.input.next() {
            let mut prev = self.prev?;
            if prev.overlaps(&i) {
                prev = prev.merge(&i);
                self.prev = Some(prev);
            } else {
                self.prev = Some(i);
                return Some(prev);
            }
        }
        self.prev.take()
    }
}

fn merge_overlapping_intervals(mut input: impl Iterator<Item = Interval>) -> impl Iterator<Item = Interval> {
    let prev = input.next();
    MergeOverlappingIntervals { input, prev }
}
```

The underlying algorithm is pretty much the same, but instead of allocating the entire result, each time `<MergeOverlappingIntervals as Iterator>::next` is called, the next element in the sequence will be returned. If you look at the amount of boiler plate and code changes needed, they are mechanical, particularly in such a simple example, but tedious.

## *“Surely, there **must** be a **better** way!”* <br/>Generators and coroutines

But what if we *didn't* have to write `Iterator` `impl`ementations by hand? Other languages, [like Python][Python generators], have "Generators" (or [coroutines] for the asynchronous version) as part of their syntax, letting you define an Item that *looks* like a regular function, but are instead "resumable functions" that instead of returning a single value use the `yield` keyword to specify a point where execution must stop and the next element should be given back to the caller. It is a mechanism to *implicitly* build state machines and represent their state transition using the `yield` keyword. This lets people write allocating code and then look for every place where a `ret.push(val)` occurs and replace it with `yield val`.

I believe such a feature in Rust would be desirable. This might not give you the *full* flexibility of writing `impl Iterator` by hand, but I think it would cover the vast majority of cases people *care* about and are now stopped by the sudden complexity hurdle.

What if in Rust we could write something along the lines of the following?

```rust
fn merge_overlapping_intervals(input: impl Iterator<Interval>) -> impl Iterator<Item = Interval> {
    let mut prev = input.next()?;
    for i in input {
        if prev.overlaps(&i) {
            prev = prev.merge(&i);
        } else {
            yield prev;
            prev = i;
        }
    }
    yield prev;
}
```


[Iterator pattern]: https://en.wikipedia.org/wiki/Iterator_pattern
[Iterator impl]: https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=841ff926cef0f0dade57b7f0184c6783
[Python generators]: https://diveintopython3.problemsolving.io/generators.html#generators
[coroutines]: https://docs.python.org/3/library/asyncio-task.html#coroutines

## Generators in Rust

As it happens, nightly Rust *has* a [Generators] feature that allows `rustc` to construct a "resumable function". They even are [the underlying mechanism that makes `async`/`await` work][tmandry async/await]. The way `rustc` desugars them is not dissimilar to what is done for other closures: they get [compiled down to a struct][state machine] that contain a field for each binding that will cross a `yield` point.

It currently can only be used in closures. When a `yield` statement is in a closure, this turns it into a generator instead, subtly changing its behavior.

The feature tries to be generally useful, which require a lot of... sometimes esoteric behavior, and the questions around the semantics for the entire feature is one of the reasons that generators are not yet in stable Rust. For example, if we look at the [`Generator` trait definition][trait Generator], we can see that there are four elements of "interactivity":

```rust
pub trait Generator<R = ()> {
    type Yield;
    type Return;
    fn resume(
        self: Pin<&mut Self>,
        arg: R
    ) -> GeneratorState<Self::Yield, Self::Return>;
}
```

First we are likely to notice the two associated types: `Yield` and `Return`. The first is the type being returned on every invocation of `yield`. If `type Yield = i32;`, then `yield 42;` is valid, while `yield "";` wouldn't be. Then we have `Return`, which is how you can customize the return value of the `Generator` closure. This is one of the, in my opinion, esoteric features of generators: being able to interact with a `Generator` that has a non-`()` return type would require new syntax on the *consumer* side. But [I believe the most common use case for `Generator`s is to implement an `Iterator` or `Stream`][why impl?], which only has *one* relevant type: the type that is yielded on each iteration:

```rust
trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}
```

Another element from the `Generator` trait you can observe is the type argument `R`, which by default is `()`. This type argument is then used in `resume`. It let's us pass information *into* the `Generator` and modify its behavior on the next iteration based on what is passed in. This would be useful for implementing a Lexer or Parser in terms of a generator, for example. But doing this is in a satisfactory manner would require new consumer-side syntax, for the vast majority of use-cases I believe these are not necessary, *and* we can design things so that whatever we do *isn't* a one way door. We can stabilize a subset of the feature *without* closing the door to extensions.

The final difference you can see is that the `resume` function takes `Pin<&mut self>`, instead of `&mut self` like `Iterator` does. This means that some bridging is needed to interface `G: Generator` and `I: Iterator`, we can't just `impl<G: Generator> Iterator for G`. But we *can* write a new-type that gets us there:

```rust
pub struct IterGen<G: Generator<Return = ()> + Unpin>(G);

impl<G: Generator<Return = ()> + Unpin> Iterator for IterGen<G> {
    type Item = G::Yield;

    fn next(&mut self) -> Option<Self::Item> {
        match Pin::new(&mut self.0).resume(()) {
            GeneratorState::Yielded(item) => Some(item),
            GeneratorState::Complete(()) => None,
        }
    }
}

```

Because of all of this, I want to focus on providing a way to create `Iterator`s and `Stream`s *without* writing the necessary boilerplate. Accomplishing this is [one of the focus areas of the Async Foundations Working Group][wg-async roadmap].

[Generators]: https://doc.rust-lang.org/beta/unstable-book/language-features/generators.html
[state machine]: https://doc.rust-lang.org/beta/unstable-book/language-features/generators.html#generators-as-state-machines
[tmandry async/await]: https://tmandry.gitlab.io/blog/posts/optimizing-await-1/
[trait Generator]: https://doc.rust-lang.org/stable/std/ops/trait.Generator.html
[wg-async roadmap]: https://rust-lang.github.io/wg-async-foundations/vision/roadmap.html
[why impl?]: https://twitter.com/ekuber/status/1392608787568029703

## Going on a tangent <br/><small>`std::iter::from_fn` and friends</small>

There's one elephant in the room when talking about generators in Rust: [`from_fn`]. This is a function that takes a closure which will get called every time the `Iterator` built by the function call gets called. This function is *very* useful. It takes a single argument, a closure, and returns an iterator that will call that closure that will get called every time `Iterator::next` on it.

You can represent [the solution we've talked about in terms of `from_fn`][from_fn solution]:

```rust
fn merge_overlapping_intervals(mut input: impl Iterator<Item = Interval>) -> impl Iterator<Item = Interval> {
    let mut prev = input.next();
    std::iter::from_fn(move || {
        while let Some(i) = input.next() {
            let p = prev?;
            if p.overlaps(&i) {
                prev = Some(p.merge(&i));
            } else {
                prev = Some(i);
                return Some(p);
            }
        }
        prev.take()
    })
}
```

But as you can notice, it has a caveat: the algorithm has to be changed in the same way that the ["`impl Iterator` by hand" version][Iterator impl] had: you must express this logic in terms of `while let` loops instead of `for`-loops. I see this is as a slightly less verbose version of the hand-made version. I think this is a strong restrictions that limits its discoverability, but does limit the *need* for changes to the language. Another restriction is that when using `from_fn` you have *no* control over the specialization of the resulting iterator. You get `impl Iterator` and you'll like it. If you need a custom `Iterator::size_hint`, or a `DoubleEndedIterator`, you are out of luck and have to go write the "by hand" version.

There also exist a number of crates out there that aim to address this space, amongst them [propane] (nightly only), [async_stream] (non-zero cost but usable in stable) and [genawaiter].

[from_fn]: https://doc.rust-lang.org/std/iter/fn.from_fn.html
[from_fn solution]: https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=61dcb27ea7586be9d8d5d715cd9869de
[non-returnable iter]: https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=313837fac18a28bb3568b501819bb763
[state-free]: https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=b0c683aa13b251e01b625aaf6c457040
[internal mutability]: https://doc.rust-lang.org/book/ch15-05-interior-mutability.html
[propane]: https://github.com/withoutboats/propane
[async_stream]: https://github.com/tokio-rs/async-stream
[genawaiter]: https://github.com/whatisaphone/genawaiter

## Proposal: Iterator Items

Because I want to focus on providing a way to implement `Iterator` and `Stream`, I think the right way to do so with with a new top-level item element that looks like a function, but that has the `yield` keyword in them. Languages like Python do not make this destinction in the function signature. Instead, a regular function having a `yield` statement turns it into a generator. I think such an approach is wrong for Rust, despite that being the approach taken for the nightly feature, so the signature of this Item *should* diverge from regular functions.

I have a straw-man syntax that I've been using that has the item signature that follows:

```rust
pub async fn* name(/* args */) yields Type { /* ... */ }
```

I want to hightlight once more how *easy* it would be to go from an O(n) space solution to O(1) with this feature:

```diff
-fn merge_overlapping_intervals(input: Vec<Interval>) -> Vec<Interval> {
+fn* merge_overlapping_intervals(input: impl Iterator<Item = Interval>) yields Interval {
-    let mut result = Vec::with_capacity(input.len());
-    if input.is_empty() {
-        return result;
-    }
-    let mut prev = input[0];
+    let mut prev = input.next()?;
     for i in input {
         if prev.overlaps(&i) {
             prev = prev.merge(&i);
         } else {
-            result.push(prev);
+            yield prev;
             prev = i;
         }
     }
-    result.push(prev);
+    yield prev;
-    result
 }
```

There's another benefit that should not be discounted: changing an implementation from `Iterator` to `Stream` to [make it asynchronous becomes *much* easier][async impl].

```diff
-fn* merge_overlapping_intervals(
-    input: impl Iterator<Item = Interval>,
+async fn* merge_overlapping_intervals(
+    input: impl Stream<Item = Interval>,
 ) yields Interval {
+    let mut input = Box::pin(input); // Hello `Box` my old friend. I'm here to `pin` you again 🎶
-    let mut prev = input.next()?;
+    let mut prev = input.next().await?;
-    for i in input {
+    while let Some(i) = input.next().await { // We might want special for iterating a `Stream`
         if prev.overlaps(&i) {
             prev = prev.merge(&i);
         } else {
             yield prev;
             prev = i;
         }
     }
     yield prev;
 }
```

I also think that the final feature *must* be part of the language, not because it can't be done otherwise (maybe with more effort), but because there are lots of UX niceties that can come with it. Namely, the diagnostic errors that `rustc` provides can be *much* nicer. rust-analyzer could easily let you transform from the iterator item syntax to expanded `struct` and `impl` blocks for when you need full control. People who come from other languages that *expect* `yield` to do something won't get weird errors talking about "nightly".

## Request For Help <br/><small>🗺 Lets go exploring!</small>

One of the problems I'm facing is that when introducing a new item to the syntax, it immediately turns into a bike-shed discussion. No clear "right" answer can be arrived to, and opinions fly right and left.

To avoid that, which can be unpleasant for everyone involved, I'm following a suggestion from Niko: I've [published a nightly-only proc-macro crate][iterator_item] (heavily inspired by Propane once I hit some limitations on my understanding of `async`/`await`) that provides a crate that allows you to write [the following]:

```rust
fn* merge_overlapping_intervals(
    mut input: impl Iterator<Item = Interval>,
) yields Interval {
    let mut prev = input.next()?;
    for i in input {
        if prev.overlaps(&i) {
            prev = prev.merge(&i);
        } else {
            yield prev;
            prev = i;
        }
    }
    yield prev;
}
```

Is everyone going to like that syntax? Very much no. I have to admit that I originally settled on this one as a *joke*, but it's grown on me and I quite enjoy it now. But of course not everyone will agree. That's where *you* come in. [This is implemented in a proc-macro][iterator_item] instead of a patch to `rustc` to make it *easier* to hack on it. If you have ✨ideas✨ about what the syntax *should* be, then you can easily [fork this crate][iterator_item] and implement it! [Show me][PRs] what you think this should look like! There are no assurances that your preferred syntax will be accepted (not even people in the Language Team can expect that), but having a way to try these out in the wild does increase the likelihood of the final syntax being one that is battle tested and not as controversial as it otherwise would be.

<small>
<em>What about the semantics?</em>

I'll have a follow up blogpost detailing the desired semantics and the outstanding open questions.
</small>

I leave you with homework: try your hand at coming up with a nice syntax for iterator items. The [tests] have some good examples of what it would look like to actually use these in practice. If you have other examples you think would be informative, feel free to open PRs to add them. I've made an effort to make the codebase well documented and easy to follow. If that is not the case, [file tickets].

For now, I'm just looking forward to a PR that makes me go "*Oh! I hadn't thought of that. I like it!*"

[Will it be yours?][PRs]


[iterator_item]: https://github.com/estebank/iterator_item
[file tickets]: https://github.com/estebank/iterator_item/issues
[PRs]: https://github.com/estebank/iterator_item/pulls
[the following]: https://github.com/withoutboats/propane/blob/00f07b3b3f3807e12b7b75e39ea251d023d9c390/src/lib.rs#L139
[tests]: https://github.com/estebank/iterator_item/tree/master/tests
[merge overlapping intervals]: https://www.geeksforgeeks.org/merging-intervals/
[async impl]: https://github.com/estebank/iterator_item/blob/e125b300d50cbe0e74a2de8d3bc732162433a10b/tests/merge_overlapping_intervals.rs#L287-L305
