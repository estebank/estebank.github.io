# No telemetry in the Rust compiler: metrics without betraying user privacy

*Esteban Küber, 2023-08-01*

**Trust is easily lost, and difficult to recover.** The Rust community trusts us, its maintainers, to ship reliable tooling. Users also trust us to respect their privacy. In order to maintain the reliability of our tooling, we mainly rely on our users' feedback, but this leaves us with limited visibility into the impact of changes we've made to the compiler.

A proposal that often comes up is the integration of telemetry into user software. I believe that no compiler needs to call home. But I also believe that we have a viable alternative if we reject the *tele* part of *telemetry*. Local metrics that are never sent anywhere can still give us valuable insight about how the Rust compiler is behaving.

## Telemetry

Telemetry is any mechanism that allows software to collect information about its own execution and/or its environment (*metron* means *to measure*), and *sends it somewhere else* (hence *tele*, which means *remote*). This is a common practice in back-end services where software development, deployment and execution all occur under a single "roof", a single organization that is already "trusted"[^remote-trust] to have control of any data that the application operates on. Proprietary software in users' machines sometimes also include telemetry, with different levels of intrusiveness. In other spaces, particularly for Free/Libre and Open Source Software and developer tools in general, the idea of any kind of communication between an application running on a user's machine and a remote server is viewed with suspicion, if not out-right disdain.

[^remote-trust]: I'm putting this in quotes because that trust is in many cases implicit, and some users either don't know about the scope of the data being collected, or they know and dislike it but there are reasons that do not allow them to use alternatives.

I understand developers' desire to have increased visibility into how their software is running in real environments. People who have worked on distributed web services know how useful it is to have full visibility on how their applications work in production (I am one of them!). This is why people with experience in distributed web services often propose telemetry in consumer software; telemetry (commonly referred-to as "observability" in this space) is one of the first things to be added to a new service, even before it is able to do anything useful.

Consumer software is different. Users have an expectation that software will do only what it "says on the tin", not mess with their data in unwanted ways, including sending it to a third party. The relationship between developers and users is then in tension. **Developers want to know as much as possible in order to service their users, and users don't want to be worried about their applications working against their interests**. A consequence of this dynamic is that when telemetry is proposed, it is immediately perceived by users as the software they trusted starting to spy on them.[^spying]

[^spying]: This perception has root both in culture (FLOSS users expect full control over their software and hardware) and in history (we could find plenty of examples of shady uses of telemetry, with different degrees of malice).

In the context of Rust, I believe that we must ensure we deliver the level of reliability that we want, and for that we need to have visibility into how the Rust tooling is working for people. Our current approach has worked for us so far, but it leaves me uneasy. Whenever we release a new compiler version, every 6 weeks, there is always a degree of "faith" that we haven't missed any regressions. But we do make mistakes and I want to figure out how we can avoid those in the future. **I've been thinking about whether we could allow the project to collect useful data from interested parties without impacting the community’s privacy**[^privacy]. User trust is key. Obviously, I would get a lot of useful information if I were to always be looking over someone's shoulder when a compiler bug occurs. But if I were doing that, the user's most likely reaction would be *"Who the hell are you and how did you get into my house!?"*

[^privacy]: Although I am part of the [Rust Compiler Team][t-compiler], and employed at AWS to work on Rust, all the contents of this post are my own thoughts on the matter. I have desired to have additional visibility into how our tooling works for a *long* time now.

[t-compiler]: https://www.rust-lang.org/governance/teams/compiler

Having telemetry in a compiler is a contentious issue, and even with all the good intent in the world, the capacity for misuse, even unintentional, is too great. Because of this **I'm convinced that `rustc` should *never* have the capability to make network connections**. Such a feature would be immediately felt as a betrayal of our users' trust. I would *love* to have omniscient knowledge of how the Rust tools work on users' machines, but implementing such machinery would go against the stated and implicit goals of the project. **When it comes to Rust, users' machines are (and should be) entirely under the users' control**.

This post is my attempt to scope out how we could close the "reliability gap", allow people in the project to have visibility into the behavior of the software we write, to help us in our release goals, without breaking our users' trust.

## Current state

Since its first stable release, the Rust Programming Language has had a commitment to [stability] that assures its users that any code that compiles on release 1.N will work on every subsequent release[^stability]. In order to accomplish this, the Rust Release, Library and Compiler teams rely on the [Crater tool][crater], an automated service that builds a given compiler version from source code and then uses it to build and run the tests of every publicly accessible Rust crate in crates.io. The result of those builds are compared against previous runs in order to detect when a regression has been introduced. We also have a similar tool to track performance regressions of the generated code called [`rustc-perf`][perf].

[crater]: https://rustc-dev-guide.rust-lang.org/tests/crater.html
[perf]: https://rustc-dev-guide.rust-lang.org/tests/perf.html
[stability]: https://blog.rust-lang.org/2014/10/30/Stability.html
[^stability]: With the only explicit carve-out of fixes to unsoundness in the compiler.

`rustc` today has some flags that you can use to [understand where your compile times are going][slow-build]. These are sometimes used by people in the compiler team during development, particularly when experimenting with alternatives to speed `rustc` up. The compiler also uses [the `tracing` crate][tracing] for logging. By default `rustc` itself gets compiled with minimal logging, clamping the level at `INFO`, avoiding the performance impact of emitting `DEBUG` logging information. All of these are useful tools available to `rustc` developers.

[slow-build]: https://fasterthanli.me/articles/why-is-my-rust-build-so-slow#how-much-time-are-we-spending-on-these-steps
[tracing]: https://crates.io/crates/tracing/

All of this, coupled with fuzzing, manual testing of nightly features during development, and a conservative stabilization process, allows the project to detect problems early and minimize the likelihood of botching a stable release. But this process isn’t perfect, as evidenced by the 27 dot-releases fixing issues in our scheduled releases. Some of these cases are really minor, i.e., fixing bugs that will go unnoticed by  most people, while others have substantial impact. There is another category of bugs for which we lack visibility:  bugs that can only be caused by incomplete code or code that is being modified, which only ever exists in a transient manner during development in users’ machines, and bugs that can only evidence themselves on uncommon platforms.
 
As an example of this, [“strict incremental fingerprint checking” was enabled in the 1.52 release][strict-incremental] by default, which would find cases of metadata corruption when using incremental builds. As part of the hardening of the incremental compilation feature, this fingerprint checking had been enabled by default only in nightly and beta. After a few release cycles, the compiler team had high certainty of having handled all outstanding issues, so a stable release was published with this check enabled by default. Once this release reached users, new unaddressed issues were encountered. An [emergency dot-release was published][1.52.1] disabling the underlying incremental compilation feature while the Compiler team worked on fixing the outstanding issues. 

[strict-incremental]: https://github.com/rust-lang/rust/pull/83007
[1.52.1]: https://blog.rust-lang.org/2021/05/10/Rust-1.52.1.html

People saw ICEs[^ICE] on nightly and didn't report them. Maybe they thought they were already fixed, maybe they didn't know how to report them. Whatever the reason, nightly and beta failed to be an effective canary.

[^ICE]: An ICE, *Internal Compiler Error*, is what we call when `rustc` itself `panic`s and has to stop in an unexpected way. They should never happen. 👀

In a world where we had telemetry in place, we would have had an additional mechanism to prevent this. Instead of enabling fingerprinting by default, the compiler team would have instrumented whether fingerprint failures had occurred across all release channels, while avoiding presenting ICEs to the user, to gauge stability of the feature without any user impact.

## Metrics

As I have already stated, I do not think we should have telemetry in the compiler. I think **`rustc` should emit metrics locally**, configurable by users through `rustup`. All of our tools should be instrumented with counters and minimal information about the environment the application has run on (like platform and compiler version). We would only add metrics as a way to answer a specific question in mind, with an explicit documented rationale. All of this information would only be stored to disk, with some minimal retention policy to avoid wasteful use of users’ hard drives.

This would allow the Rust Release Team to use these metrics to increase the visibility into changes when running Crater and Perf, by making these two services consume these metrics and collate them for us. A separate tool, or even the end users themselves manually, would be responsible to upload these metrics to the project if they so desire as part of specific tickets. Without user action the data would never go anywhere. This separation of concerns would reduce both the perception and reality of potential nefarious use of any collected information, as there would be no centralized collection to speak of.

I could picture a future where the Compiler Team realizes we have an important question about the behavior of the software, that could be answered by analyzing these metrics, and have a VSCode or cargo plugin that analyzes these metrics *locally*, and if relevant information is found in these logs an appropriate summary is presented to the user for them to file a ticket. Custom tools could be developed to leverage this information, such as a linter that uses these metrics to suggest changes to your codebase to make it compile faster or produce a smaller binary. Organizations that manage their developers' environments could use this information to improve their processes and upstream compilation performance improvements that they are affected by. This kind of data would help people working on the compiler to make it faster.

I believe enabling these use cases is essential to maintain both development velocity and high reliability in the project. It allows us to address questions from real production environments while still preserving our users' privacy.

I wrote this piece for multiple reasons. I want to draw a line in the sand for ourselves, a clear statement of intent that **user information should never leave their machine in an automated manner**, even by accident. *Any* automated mechanism would have to clear a high bar that the user is aware of the behavior and enthusiastically participating. I want to hear from the community[^comms] about any issues that would have to be mitigated when introducing locally stored compiler metrics. And I don't want people to be concerned if the Compiler and Release teams start talking about and implementing a metrics mechanism by confusing their scope with some kind of end-user telemetry.

[^comms]: There are multiple places where this post might have been linked from, and I will try to keep on top of each and evey one of them. If you want to ensure I see your feedback, consider joining the conversation at [internals.rust-lang.org](https://internals.rust-lang.org/t/no-telemetry-in-the-rust-compiler-metrics-without-betraying-user-privacy/19275) or emailing me at metrics@kuber.com.ar.
