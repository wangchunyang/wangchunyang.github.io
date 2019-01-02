---
layout: post
title: "Java Raw String Literals -- restarting the discussion"
---

# Java Raw String Literals -- restarting the discussion

## Original link:

https://mail.openjdk.java.net/pipermail/amber-spec-experts/2019-January/000931.html

## Content

As many of you saw, we pulled back the Raw String Literals feature from JDK 12.  The public statement is here:

   http://mail.openjdk.java.net/pipermail/jdk-dev/2018-December/002402.html

So, let's restart the design discussion.  First, I want to enumerate some of the process errors I think we made.  

 - We never really explored the full design space.  The initial proposal had a reasonable syntactic strawman, and rather than explore the entire space, we mostly followed the path of refining the initial strawman, and stopped there.  
 - We got caught in the "linear thinking" trap with respect to the design center.  We started off thinking of this feature as "raw strings", of which multi-line strings are an important sub-case, but in reality most of the user pain is over dealing with multi-line snippets of HTML, JSON, XML, or SQL, and raw-ness is secondary.  We never really made this turn.
 - We were too focused on getting the last 2% rather than the first 98%.  (Note that for many, perhaps most language features, the last 2% is critical; for this one, which is entirely about syntactic convenience, it is not.)  
 Specifically, by focusing on self-embedding as a test of fitness rather than more typical use cases, we ended up in a place that was both more complex than necessary, and at the same time, still had prominent anomalies.  (Anomalies are unavoidable if we are unwilling to take on a super-ugly syntax, but we do have some control over how obvious and prominent they are.)

From my "language steward" perspective, my main problem is that the two forms of string literals in the current proposal are gratuitously unrelated.  They are syntactically unrelated (different delimiters and delimiter arity rules), and semantically unrelated (one must be raw and permits multiple lines; the other cannot be raw and cannot be multiple line.)  I would prefer to have a single string literal feature, with some sub-options for controlling raw-ness and/or line spanning -- with bonus points if these are orthogonal aspects.  (As a sub-concern, I would strongly prefer we not burn the backtick character as a delimiter; it should be entirely possible to avoid this by building on the existing string literal mechanism.)  

So, how should we evaluate success here?  This feature doesn't improve the expressiveness or abstractive ability of the language at all -- it's purely about syntactic convenience.  And, given that we've limped along for 20+ years without it, it's lack can't be all _that_ problematic.  So let's identify the use cases we care about most, and evaluate the feature through the lens of how it helps those use cases.  In my opinion, these are:

 - Multi-line snippets of JSON, HTML, XML, and SQL embedded in Java code as string literals. (Other languages are used too, but these constitute the majority.)  These currently require escaping for quotes and for newlines, which means every such snippet requires substantial surgery.  This is painful for code writers (though IDEs can do most of the lifting here), but more importantly, is harder to read, and it is really easy to leave out a `\n` and get the wrong result, and not have it be immediately noticeable.  We would like for most such snippets to be simply pastable without modification.  
 - Regular expressions and Windows paths routinely require escaping, which again is easy to get wrong and hard to read.  (Regular expressions are hard enough to read, we don't need to make it harder.)  These are typically a single line.  

Given that this feature is pure convenience, we'd also like to avoid excessive spending of our complexity budgets -- either language complexity or teachability.  Grabbing for that last 2% at the expense of either of these is not a good trade.  

Note too that there is no ideal answer here; we can see this quite clearly by looking at the variety of choices other languages have made, and each still has anomalies (e.g., python raw strings can't end with a backslash) or forces ugly complexity on the reader (e.g., user-selected nonces in C++ raw strings, or Rust's `#` characters).  This is truly a "pick your poison" game.  

Let's remind ourselves of what other languages do in this area.  In all these languages, raw strings can contain newlines; some have separate features for multi-line escaped strings and multi-line raw strings.  

 - C simulates multi-line strings by having a continuation character (backslash) in the last column, or by implicitly concatenating adjacent string literals (`"raw" "string"`).  It does not support raw strings, though there is a gcc extension that emulates C++ raw strings.  
 - C++ supports multi-line strings through raw strings.  It denotes raw strings with an `R` prefix before the quotes, and a user-selected nonce and parentheses inside the quotes: `R"NONCE(raw string)NONCE"`.  The nonce may be empty, but the parens are required.  
 - Rust supports multi-line strings by simply allowing newline characters in an ordinary string literal.  It separately supports raw string literals with an `r` prefix, followed by a variable (can be zero) number of `#` characters, a double quote, the raw string, a double quote, and the same number of `#` characters: `r##"raw string"##`.  
 - Python allows string literals to span multiple lines by using a three-quote (`"""`) delimiter.  It allows raw string literals by prefixing the string literal with `r`.  Its escaping rules for quotes in raw strings are unusual; a backslash preceded by a quote escapes the quote, but leaves the backspace in the string.  (Accordingly, a raw string cannot end with a backslash.)
 - Ruby supports multi-line strings with here-docs, and raw strings using the `%q()` construct: `q(raw string)`.  
 - C#, like C++, support multi-line strings through raw strings.  A raw string precedes the string literal with an `@` character: `@"raw string"`.
 - Scala and Kotlin, like C++ and C#, support multi-line strings through raw strings.  A raw string is delimited with triple quotes: `"""raw string"""`.

Note too that there is also room for interpretation on the meaning of "raw"; Python permits some escaping in raw strings, and Kotlin permit interpolation in raw strings.  

We can divide the approaches roughly into three categories:
 - Those that use user-supplied nonces (C++, here-docs).  These can render 100% of embedded strings, with the costs that come with nonces: annoying to write, and imposing cognitive load to read (as nearly any sequence can be a nonce.)
 - Those that use variable-sized delimiters (Rust, and our previous proposal).  These are simpler, but will invariably have some anomalies.  
 - Those that use fixed delimiters (C#, Scala).  These are simpler still, and will have more anomalies.

So, recapping our starting point and guidance:

 - The primarily use case is multi-line snippets of JSON, HTML, XML, and SQL.  It is rare that these require true-raw-ness, but they all commonly have embedded quote characters.
 - The secondary use case is truly raw strings, of which the most common offenders are small-ish -- regular expressions and windows paths.  
 - We should start by trying to extend existing string literals to support raw and/or multi-line strings.

Some questions we need to answer:

 - What are reasonable delimiter choices for raw and/or multi-line strings?
 - Should the default treatment of multi-line strings be raw or escaped (alternately, is this one feature or two)?
 - Is raw-ness a property of a string literal, or a state that can change within the literal (i.e., with embedded start-raw/end-raw escape sequences)?
 - How do we embed delimiters in raw strings (escaping, doubling up, concatenation)?
 - How far do we want to go to support embedding of delimiters?  

Let's start by asking how we might extend the current string literal feature to support multi-line strings.  Currently, a string literal starts with a double-quote, can span only a single line of source, and ends at the first unescaped double quote.  How could we extend this to a multi-line string literal?  Some possibilities include:

 - Simply remove the constraint of "can only span a single line"; no other change to delimiters is required (the Rust approach.)  
 - Choose a different fixed delimiter, such as tripled quotes ("""),  doubled single-quotes (''...''), or a multi-character quote token (`/"..."/`).
 - Use a modifier on the opening quote, such as `R"..."` or `@"..."`
 - Use an embedded escape sequence, such as `"\M..."`, to opt into multi-line treatment
 - Use here-docs, with a fixed or user-providable nonce

I think its reasonable to eliminate here-docs from consideration as these are more typically associated with scripting languages.  

At first blush, the simplicity of the Rust approach is attractive; just let strings span multiple lines, with no new syntax.  The obvious counter-arguments are pretty weak in the current age; if you code in IDE, as most developers do, it is not easy to accidentally leave off a closing quote, and the syntax highlighting will make this obvious in the event we do so anyway.  But, if we look through the lens of our use cases -- such as JSON snippets -- we see that this approach fails almost completely, because you _still_ have to escape the quotes, and almost all multi-line snippets will have quotes.  So, let's cross this off too.  The same applies to using a letter prefix for multi-line strings; it doesn't address the primary use case.  

Note too that our primary use case admits a middle-ground option: multi-line strings are not raw, but quotes need not be escaped.  This is a possibility if the delimiter is anything other than a single double-quote (`"`).  

So, some reasonable starting points on this front include:

 - Just follow C#/Scala/Kotlin, where there's a single mechanism for both raw and multi-line, delimited by triple-quotes.  Here, a single (or double) embedded quote does not necessarily need to be escaped.  
 - Use triple-quotes for non-raw multi-line string literals, and some sort of additional way to select raw-ness for either single- or triple-quoted string literals.  (Same comment about embedded quotes.)  
 - Same, but use doubled or tripled single-quotes.  

Within the "multiple quote" options, we can separately choose between a fixed number of quotes (e.g., 3) or a variable number (e.g., 3 or more, odd only, etc.)  The trade-off here is about where the anomalies go; with the variable-number approaches, it gets harder to start or end with the delimiter character (while this is not necessarily a serious anomaly, but it is a prominent one), and with the fixed approach, there is more need to do something (escaping, concatenating, etc) the delimiter character (though embedding triple-quotes is not all that common in our primary use cases).  Also, our IDE friends have pointed out that even numbers of quotes put the IDE in a quandary as to whether the user has just typed the opening delimiter, or both the opening and closing delimiters.

Now, raw-ness.  

One option is to just say that multi-line strings are also raw.  We have evidence that this is not totally unworkable, as several languages have gone this way, but it does mean that for the use cases where the user wants multi-line but not raw, they must resort either to concatenation, or explicit escape processing (e.g., `"""foo""".escape()`)

Another is to allow a prefix character to indicate raw-ness; `R"foo"` or `R"""foo"""`.  The prefix character approach is more extensible to other kinds of modes to string processing.  

Another option is to use a different delimiter, as the current proposal does.  If we were to go this way, I'd suggest we consider double or triple single-quote (which are currently illegal), rather than continuing with backtick.  

A fourth option, one that has not yet been considered, is to say that raw-ness is a _state_ of processing a string literal; string literals start out escaped, but can drop into (and out of) raw-ness as they like:

    String s = "This part is escaped\n, but this part\- is raw, and this part\+ is escaped again."
    String path = "\-C:\bin\putty";

This gets us where multi-line-ness and raw-ness are orthogonal properties of string literals -- without requiring any new delimiters.

So, how to proceed?  First, let's try to avoid focusing on our own personal preferences, or be distracted by unfamiliarity, and remember that our job here is to get to a design that's best for _tomorrow's_ Java developers and source base.  (That means that, for example, we can't allow ourselves to be distracted by the fact that, say, embedded "\-" or `R"..."` is unfamiliar today.  It will be familiar tomorrow, if we decide that's what would be best.)

Here's what would be super-useful:

 - Data that supports or refutes the claim that our primary use cases are embedded JSON, HTML, XML, and SQL.  
 - Use cases we've left out, for which we can discuss whether we want to incorporate them into our goals.
 - Data (either Java or non-Java) on the use of various flavors of strings (raw, multi-line, etc) in real codebases, which might be useful to help determine, for example, whether raw and multi-line should be lumped into the same bucket or not.  

The bike shed is open (but please show up with structural members, not just paint.)
