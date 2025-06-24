+++
title = "Sample code warnings, and forgetting who's who"
description = "A small but mildly interesting UI design problem, which revealed an odd mental block"
date = 2025-06-24

[taxonomies]
tags = ["meta", "css"]
+++

This post describes a small but enjoyably twisty UI design problem, with some musings on how we sometimes make life unnecessarily difficult for ourselves.

Like most nerdily inclined blogs, this one will often contain code blocks. Since it's learning-adjacent for now, a fair few of those code blocks will have something wrong with them. It seems only fair to warn the reader of this, prominently, and this is very well-trodden ground in technical writing. The world has converged on some sort of overlay badge for the code block; anyone who's looked at the Rust Book, for example, will have seen its three variations on [Unhappy Ferris](https://doc.rust-lang.org/book/ch00-00-introduction.html#ferris).

Why do this at all? What's wrong with a comment? The general consensus seems to be that comments aren't prominent enough. This is especially true for some syntax-highlighting colour schemes which try to make comments as invisible as possible -- I'm looking at you, Dracula. Now, one might reasonably object that we could just not choose those schemes, but I think it points to a deeper issue: that some people don't like comments and don't generally read them. Maybe they've been burned too many times by out-of-date or misleading comments, or maybe they've had to read too many variations on `i += 1; // add 1 to i`. That's their prerogative, but "this code will crash your program" feels important enough to trump personal distaste.

{% details(summary="This can happen with error messages too") %}
In my last job while working on a search API, I'd sometimes get plaintive messages from users whose requests weren't working. It was remarkable how often the lovingly-crafted error message they were getting specifically pointed to the problem and how to fix it. But they'd been conditioned to assume that error messages never contain actionable information for users, and so there was no point reading them. I couldn't fault them. Most of the time it's true. In a visual interface you can mitigate this by styling the error page to look intentional and nonthreatening, but there's limited scope for that in JSON.
{% end %}

## The basics

So far, so straightforward. Personally I'm not wild about graphical badges -- I don't find the individual Ferrises very recognizable, and language-specific themes like the crab don't work so well in a multiple-language environment -- but the general idea is the same. I'm using Markdown and Zola, which translates a code block to a `<pre>` containing a `<code>` containing a bunch of individually-styled `<span>`s. So we just need to tag our fenced code block to indicate that it needs a badge, declare a CSS pseudoelement positioned over it, style it to taste, and saunter home for tea and medals.

{% details(summary="How do we tag it?") %}
CommonMark's fenced (triple-backtick) code blocks support what it calls an [info string](https://spec.commonmark.org/0.28/#fenced-code-blocks) after the opening fence. The first word of this is conventionally the language, which will be used by the highlighter. Treatment of the rest is undefined by the spec, but processors commonly support other parameters. Zola doesn't have one for warnings, but it does have a fairly generic-looking one for `name` which we can abuse here; Zola itself doesn't do anything with it except pass it through as a `data-name` attribute.
{% end %}

```css
pre[data-name="invalid"] { position: relative; }
pre[data-name="invalid"] code::before { 
    content: "Invalid code!"; display: inline-block; position: absolute; right: 0; top: 0; 
    /* whatever cosmetic styling tickles your fancy */
}
```

{% details(summary="Hey, you're hardcoding the badge message!") %}
Yes, this is a very reasonable objection. Of course, we could pass an arbitrary string as the value of `data-name` and use that as the `content`, but that would complicate our CSS selector; `name` might be used for other purposes so treating any value as a warning feels wrong. Hardcoding does make things a bit bland if we're trying to stay language-neutral: you don't want "does not compile" on a JavaScript block, or "may panic" on CSS. We could define multiple possible warning values for `data-name`, with different `content` for each, but that doesn't really scale. Don't worry, we'll come back to this later.
{% end %}

This isn't too bad as far as it goes. One issue that pops up early is that the badge can obscure the end of the first line or two of code; this is especially true of Rust where the first line is often a function signature and those can get very very long. Not to worry, we can add another rule for `pre[data-name="invalid"]:hover code::before` to fade the badge to zero opacity when mousing over the code block; it's fairly safe to assume they'll have read it by then. Are we done now? My tea and medals are getting cold.

## Not so fast

What about people who can't hover, maybe because they're using one of those newfangled portable telephones? They're even more vulnerable to having content obscured since their displays are so much narrower. Hmm. All right, so we'll make the top-right `position` conditional on a `@media (any-hover: hover)` query. Otherwise (`any-hover: none`) we'll set the badge to `display: block`; it'll still get styled prominently, but now it'll just sit at the top without obscuring anything, and can be scrolled along with the code. Can I...

What about screenreaders, which don't reliably read CSS-added pseudoelement content? What about RSS/Atom feeds, or text-only browsers like [Lynx](https://en.wikipedia.org/wiki/Lynx_(web_browser)), all of which ignore CSS? What about scraper bots, which may be feeding into an LLM training corpus, assuming we're not signed up to the Butlerian Jihad and actively *trying* to teach the machines to write crashy code?

OK, so we'll... erm... hm.

## Full circle

Let's step back for a moment to our early question: *why not a humble comment in the code block?* A comment can contain any text you like; you don't need to worry about how to plumb it through the Markdown processor. A comment doesn't rely on CSS to be readable. A comment is handled fine by screenreaders. (Although how screenreaders cope with Rust is another question; it may be second only to Perl in the "languages that look like Snoopy swearing" rankings.)

There's still the prominence issue which drove us away from comments in the first place. But that's perfectly soluble! Where appropriate, we can just use CSS to pick out the first line of the code block, then position and style it the same way we did before:

```css
pre[data-name="invalid"] span:first-of-type {
    display: block;
    /* style to taste again, but note that you'll probably need to use important! on
    some rules to override attribute or high-specificity styles from the highlighter */
}
@media (any-hover: hover) {
    pre[data-name="invalid"] { position: relative; }
    pre[data-name="invalid"] span:first-of-type { position: absolute; right: 0; top: 0; }
}
```

Some objections arise:

*"The badge will contain the comment syntax chars, like `// `!"* This genuinely gave me pause. Since I have this blog on a strict no-JavaScript diet, my actual response was "Dammit, maybe I could use a negative margin to push those chars outside the clipping box, or add a pseudoelement to cover them up, or...". The correct response would have been "Yes, and?" This is very much a personal-taste thing, but the moment I actually tried it, it felt... fine. It *is* a comment, just a prominent one. It does no harm to make that clear; that it isn't an error message or some sort of call-to-action button. The reader is already assumed to be comfortable reading code; comment syntax is not going to throw them into panicked flight. Ultimately, this just feels the right way around to me: a dumb reliable base, with progressive enhancement where supported and appropriate.

*"You're making assumptions about the contents of the code block!"* Yes, I am. I'm the one writing the code block.

*"You use `important!` to override the highlighter's styles, and that's bad practice!"* Fair in general, but it's very tightly scoped here. I'm not going to lose sleep over it.

{% details(summary="A couple of other minor caveats") %}
- This won't work with comment-less content like JSON. (And probably [Befunge](https://en.wikipedia.org/wiki/Befunge), since that just relies on routing control flow around the "comment", which is tough to do on line 1. Either way, it's a short list.) I'm OK with this; I can't imagine many cases where I'd want to show broken JSON, but if necessary I could always customize styling by selecting on the `data-lang` attribute.
- It might get tricky if the processor does no highlighting. Zola is fine here, still generating a span per line, but if you're using a different setup then it's worth checking.
{% end %}

The end result (including styling, no-hover and hide-on-mouseover tweaks) looks like this. (It may have changed since this post was written, of course. That yak hair grows back eventually and needs shaving again.)

```rust,name=invalid
// will move your cheese!
fn move_cheese(cheese: &mut Cheese) { 
    cheese.x += 10; 
}
```

## You do it to yourself, you do
