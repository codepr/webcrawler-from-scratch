Project setup and tools
=======================

This chapter will cover all we need to create our webcrawler, with some brief
explanations about every choice and opinions on developer tools.

Go will be the language of choice, it's a simple yet very powerful language,
empowering the user with a straight-forward set of concurrency primitives based
on cooperative behavior[^1] and sharing-by-communication model through
channels, with a battery-included standard library and a complete toolchain
that make it easy to setup new projects, test, benchmark and adoption of
third-party libraries.

If you're new to the language, it's really easy to learn and fast to become
productive with, as it's one of the traits that Go creator put the most
emphasis when they designed it.

Compared to many backend languages it's documentation and standard library is
quiet compact,
[https://golang.org/doc/effective_go.html](https://golang.org/doc/effective_go.html)
and [https://tour.golang.org/welcome/1](https://tour.golang.org/welcome/1)
provide all you need to acquire proficiency with the language.

[^1]: Opposite to pre-emptive, it's called cooperative because it's the routine that gives the control back to other routines so they can resume their execution, generally through an "orchestrator" component, like an event-loop. In Go this concurrency paradigm is spread across a pool of OS threads, this makes the so called "goroutines" really cheap to spawn (~1MB for OS threads vs ~2-5KB for each goroutine) in large number.