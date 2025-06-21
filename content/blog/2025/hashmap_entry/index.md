+++
title = "The problem with Rust's HashMap::entry"
description = "A look at performance issues with unnecessary owned key construction in Rust's HashMap::entry API"
date = 2025-06-16

[taxonomies]
tags = ["rust", "performance"]

[extra]
comment = "versions: rust 1.86.0, hashbrown 0.15.4"
+++

This is about a seemingly-minor performance detail that's niggling at me as a Rust beginner working through the Book. It can have a significant impact in some usage scenarios, it's a well-known issue, and a well-known workaround exists. It's just not available in the standard library for stable Rust, and everybody seems weirdly fine with that.

<!-- more -->

I didn't really start digging until I was a bit further along in the Book, so the post mentions some concepts (lifetimes, traits) that you might not have encountered yet if you got a bee in your bonnet at the same point I did.

## Background

The third exercise in section 8.3 involves collecting employee names grouped by department name, i.e. a `HashMap<String, Vec<String>>`. The text points to the `entry` method as the idiomatic way to add or update an entry which might not exist yet, so given a `&str` key it's straightforward. (I'm aliasing some types to reduce visual clutter.)

```rust
type V = Vec<String>;
type Map = HashMap<String, V>;

fn get_mut_or_insert_default_naive<'a>(map: &'a mut Map, k: &str) -> &'a mut V {
    map.entry(k.to_owned()).or_default()
}
```

{% details(summary="Is `String` really the right choice?") %}
When I did this exercise I was firmly of the opinion that neither department (key) nor employee (value) names should be mutable, and went with `Box<str>` instead. This has several advantages: it's guaranteed immutable even if owned by a mutable collection, it's guaranteed to be right-sized whereas `String` may have excess capacity for amortized growth, and its memory footprint is smaller by a `usize` since it doesn't need to carry a capacity separate from length. 

I'm more relaxed about `String` now; I still think `Box<str>` is better, I'm just less interested in swimming against the tide. `String` is less visually noisy, it's far more common as a basic vocabulary type, it won't over-allocate capacity unless you actually start futzing with it, and for obvious reasons `std`'s `HashMap` won't give you mutable references to its keys anyway. I'd like any solution I come up with to support `Box<str>` as well, though. (And I do wish `Box<str>` had been named `String` and `String` had been named `StringBuilder`, grumble grumble.)
{% end %}

{% details(summary="Why do we need to specify the lifetime?") %}
I was a bit surprised that I needed an explicit `'a` lifetime annotation here; I couldn't see any way a `&mut Vec<String>` return value could possibly be aliasing a `&str` parameter, so where's the ambiguity? But Rust's lifetime elision rules don't look at types. If there are two reference parameters and neither of them is `self`, you have to annotate manually. This isn't a criticism! Rust faces ongoing struggles with compilation times, a Sufficiently Smart Borrowck is not necessarily a Sufficiently Fast or Stable Borrowck, and adding the lifetime isn't too onerous.
{% end %}

This returns a possibly-new `&mut V` to which you can add a name. But `entry`'s key parameter has to be a `String`, rather than the `&str` you're holding as you parse input, and  `to_owned()` is allocating memory on the heap even in the case where the entry already exists and the new key isn't needed. This is jarring given Rust's efficiency claims, especially so early on. In the context of this exercise the overhead is irrelevant since we're being driven by interactive user input, but in a hot loop we'd want to avoid heap allocation if at all possible.

[![An XKCD cartoon modified to poke fun at premature optimization](xkcd386_duty_calls_modified.webp "“Oh no, the official XKCD script font TTF is broken! Better raise a bug.”&#013;“Wait, weren't you working through the Rust Book?”&#013;“WHAT DOES IT LOOK LIKE I'M DOING?”")](https://xkcd.com/386)

## First attempts

So... *is*  it at all possible? After all, we didn't have this problem with `get`. Why could we pass a `&str` to that but not to `entry`? 

We could pass a reference because there's no chance that a `get` call will add a new entry. A `HashMap` owns its keys, so it needs owned key values when doing that. 

We could pass a `&str` rather than a `&String` because the method signature is going out of its way to support us, typing its key parameter as a generic `&Q`, where the map's key type `K` is required to implement `Borrow<Q>`. By implementing `Borrow<str>`, the `String` type declares that its data payload is basically just a `str` and can be borrowed as a `&str`, and in particular it promises that it'll hash the same and test for equality the same as a `str` would. Perfect! In fact, the [trait docs](https://doc.rust-lang.org/std/borrow/trait.Borrow.html) for `Borrow` use `HashMap::get` as the motivating example. 

This doesn't help us with `entry`, because of the key ownership requirement. What else could we try? Well, the obvious approach would be to do a `get_mut` first, and only resort to inserting when we don't find an existing entry. Unlike `entry` we're adding an extra lookup in the insertion case, but the whole point of the optimization we're attempting is that insertions will be the exception rather than the rule. So:

```rust,name=invalid
fn get_mut_or_insert_default_probe1<'a>(map: &'a mut Map, k: &str) -> &'a mut V {
    match map.get_mut(k) {
        Some(v) => v,
        None => map.entry(k.to_owned()).or_default()
    }
}
```

This doesn't work. The borrow checker says we're trying to have two mutable references to the map at the same time. An `if let` version doesn't fare any better:

```rust,name=invalid
fn get_mut_or_insert_default_probe2<'a>(map: &'a mut Map, k: &str) -> &'a mut V {
    if let Some(v) = map.get_mut(k) {
        return v;
    }
    map.entry(k.to_owned()).or_default()
}
```

In both cases, Rust complains on the first return that *"returning this value requires that `*map` is borrowed for `'a`"*. Since it's returning a mutable reference to something in our map (indicated by the lifetime), our map reference is considered to have escaped and be roaming around at large now, and we aren't allowed to use it again here. It's frustrating because in both formulations, almost by definition, we're only trying to use it again in the branch where we *didn't* return it the first time! I don't pretend to grasp the subtleties here, but this does feel like a weakness in the borrow checker.

And, it turns out, a well-known one. This exact situation was infamous as "Problem Case #3" (which has a nice SCP ring to it) in the epic "non-lexical lifetimes" initiative running from 2017 to 2022. It was deferred from that as being too knotty, but work is ongoing to address it as part of the followup "Polonius" initiative. (I sometimes suspect that >80% of Rust engineer time is spent coming up with clever project names.) A 2024 [blog post](https://smallcultfollowing.com/babysteps/blog/2024/06/02/the-borrow-checker-within/) by Niko Matsakis reassures us that this work is still progressing.

Note that even when this work lands, it won't offer a perfect solution; there'd still be that pesky extra lookup on the insertion path. And it's not something we can do anything about for now in any case, so it's time to look elsewhere.

## Settling for mediocrity

If we abandon our doomed Utopian dream of avoiding a double lookup in the common case, we can always do this:

```rust
fn get_mut_or_insert_default_doublelookup<'a>(map: &'a mut Map, k: &str) -> &'a mut V {
    if map.contains_key(k) {
        map.get_mut(k).unwrap()
    } else {
        map.entry(k.to_owned()).or_default()
    }
}
```

This does have the minor virtue of actually compiling, but otherwise it's hard to get enthusiastic about it. It'll come down to a question of whether a usually-redundant allocation is worse than an always-redundant extra lookup. My expectation is yes (don't worry, we'll check later) but even at best it would depend heavily on the scenario. It'd be nice to have something a bit more clear-cut.

## Going outside `std`

At this point we've gone as far as we can go with the stable `std::collections::HashMap`. If we want to continue we'll need to go outside, and discussions around this problem always point to the [`hashbrown` crate](https://crates.io/crates/hashbrown). This is what `std`'s hashmap is built on, and it exposes some additional low-level `raw_entry` APIs which are tailor-made for us: the [docs](https://docs.rs/hashbrown/0.15.2/hashbrown/hash_map/struct.HashMap.html#method.raw_entry_mut) explicitly mention *"Deferring the creation of an owned key until it is known to be required"* as a use case. (Although they call that an *"exotic situation"* which struck me as odd; string-keyed maps are hardly exotic. To quote Ben Elton, that's like calling a loaf of bread "Enigma".)

Somewhat ominously, `hashbrown`'s README describes the `RawEntry` API as "deprecated", but as far as I can tell this doesn't mean anything substantive. The methods aren't marked as `#[deprecated]`, and the feature is still included by default. I suspect it's just a guardrail since stabilization failed, a way to signal that "you shouldn't assume this will ever be in `std`".

Anyway, armed with this, we can do much better:

```rust
type BrownMap = hashbrown::HashMap<String, V>;
fn get_mut_or_insert_default_hashbrown_raw<'a>(map: &'a mut BrownMap, k: &str) -> &'a mut V {
    map.raw_entry_mut()
        .from_key(k)
        .or_insert_with(|| (k.to_owned(), V::default()))
        .1 // only want the value, not the (mutable, yikes!) key
}
```

Pretty good! No redundant lookups, no redundant allocations. There must be a catch.

Obviously, it's an extra dependency that mostly just repeats what `std` already gives you. Even though `std` uses `hashbrown` itself, that doesn't make the dependency "free". `std` doesn't use Cargo to manage its own dependencies, and you aren't getting `std` via Cargo anyway, so Cargo doesn't get to apply its deduplication magic, so you'll have two copies of the `hashbrown` code in your binary. It's not a huge dep, but still.

It also doesn't help us with the `std` maps which the vast majority of Rust code will be using. And there isn't any way to get at `std`'s more fully-featured underlying map; the use of `hashbrown` is an implementation detail, and jealously guarded as such. The `raw_entry` API *is* currently exposed in nightly Rust's `std` behind a feature flag, but a [move to stabilize](https://github.com/rust-lang/rust/issues/56167) was "closed as not planned" in August 2024 so doesn't look likely to go anywhere soon. 

{% details(summary="But if `std`'s map is just a shrinkwrapper, couldn't we `transmute` a reference and...?") %}
![cat recoiling in horror](horrified.webp)

No, don't do that. Yes, it's a shrinkwrapper, in the sense that `std::collections::HashMap` is literally a struct with one member `base` of type `hashbrown::HashMap`. But you can't transmute it to the type of `base`, because you can't *see* the type of `base`. You could transmute it to the type of *your* `hashbrown::HashMap`, but `std` may not be compiling `hashbrown` with the same flags as you, and everything is going to explode horribly anyway when the versions get out of step. Or, heaven forbid, `std` switches to a different implementation entirely.
{% end %}

It's mentioned less, probably because it's newer, but `hashbrown` also offers a much more ergonomic alternative: the [`entry_ref`](https://docs.rs/hashbrown/0.15.4/hashbrown/struct.HashMap.html#method.entry_ref) API:

```rust
fn get_mut_or_insert_default_hashbrown_nice(map: &'a mut BrownMap, k: &str) -> &'a mut V {
    map.entry_ref(k).or_default()
}
```

The chained `or_default()` method here has an `Into<K>` bound on `&Q` (the key reference type) which is how it produces an owned key when needed. 

I can't find any discussion of stabilizing `entry_ref`, and that isn't exposed in nightly `std`. It isn't even in the `hashbrown` [docs](https://rust-lang.github.io/hashbrown/hashbrown/index.html) hosted under rust-lang.github.io - they're for an ancient version 0.11.2, way behind the 0.15.4 on crates.io. Is `std` just vendoring in an old snapshot and never updating?

## Does any of this actually make a difference?

Bah, do not pester me with such trifling details. 

Well then if you're going to *push* me... okay, time for some rough benchmarks. These are very crude -- I don't want to get sidetracked into Criterion.rs just yet, and maybe `cargo bench` will reach stable before I have to -- but I'm inspired by nnethercote's [advice](https://nnethercote.github.io/perf-book/benchmarking.html) that *"Mediocre benchmarking is far better than no benchmarking"*. Results are very consistent across multiple timing runs, within a millisecond or two, including with the order reversed. Standard `--release` build, `stable-x86_64-pc-windows-msvc` toolchain, AMD Ryzen 7 3700X 3.6 Ghz.

What I'm doing: outside of timing, read in a plaintext copy of Project Gutenberg's *The Complete Works of William Shakespeare* and lowercase it. Inside timing, split it into a sequence of word `&str`s using [`unicode_segmentation`](https://crates.io/crates/unicode-segmentation), and use our various `get_mut_or_insert_default_*` functions to count the number of times each word occurs. Segmentation is fairly expensive, so I also time just the word iteration loop, giving us a baseline above which hashmapping will add overhead. Sample results from each timing are printed out as sanity/consistency checks and to prevent things getting optimized away. 

This is a scenario which should flatter the experiment: each word appears an average of 34 times, so \~97% of words seen have been seen before, and the actual work done for each appearance after the first is trivial, just incrementing a counter. It's obviously not meant to represent the full range of real hashmap usage, but it's real data and doing something at least vaguely interesting. (Did you know Shakespeare never used the word "sausage"? Or "aardvark".)

| Test               | Wall time (ms) | Over baseline (ms) |
| :---               | ---:           |               ---: |
| `baseline`         | 104            | -                  |
| `std_naive`        | 200            | 96                 |
| `std_doublelookup` | 148            | 44                 |
| `hashbrown_raw`    | 118            | 14                 |
| `hashbrown_nice`   | 119            | 15                 |

That's... actually more impactful than I expected. Over twice as fast with double lookup, and nearly 7 times as fast with `raw_entry_mut` or `entry_ref`.

## TL;DR and next steps

- `HashMap::entry` can be inefficient when keys are expensive to construct and often repeated
- Obvious mitigation is foiled by the dastardly borrow checker for now
- Underwhelming mitigation is underwhelming, but not entirely useless
- `hashbrown`'s `raw_entry_mut` API brings large gains, but is not (nor likely to be) exposed in stable `std`
- `hashbrown`'s `entry_ref` API is comparably fast and nicer, but isn't even on the horizon for `std`

I'd like to try rolling these approaches up into an extension trait to complement the various owned-key methods and genericizing it to cover other applicable owned/reference type pairs, not just `String`/`&str`. (I'd actually done some of this before doing the timings above, but don't expect it to have affected the results.) This will be mainly a learning exercise in generics and trait design, and maybe a way to get my feet wet publishing to crates.io if it gets that far. This post is long enough already, though, so I'll punt that to a possible future one. I'd also like to extend the benchmark to find the insert/overwrite ratio at which `std_doublelookup` becomes worth it, and how bad it is in the worst (no overwrites at all) case. 

Yes, these are serious intentions. No, I'm definitely not just putting off the dreaded day when I get to the `async` chapter.

Regardless of outcome, it's been an educational wander around some of Rust's moving parts, especially small but important traits like `Borrow`, the relationship between `std` and its exploratory implementation crates, and some shadier corners of the borrow checker. I find this sort of magical mystery tour a much stickier way to learn than just reading or staying strictly within the bounds of a given exercise.