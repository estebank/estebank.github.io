# Sustainable growth and visibility

The Rust project and ecosystem have grown dramatically over last couple of years. Between the creation of The Rust Foundation, increasing adoption and a lot of development work by contributors of varied background, I am incredibly proud of the accomplishments and current state of the project.

The official projects have official teams and working groups, with regular meetings to coordinate self-driven contributors. This has worked great so far, but I am starting to see the limits of this way of working. These kind of contributions are *incredibly* important and welcomed, but we need alternative ways of tackling both larger feature work projects and long term maintainance. In the past, we have used the release of new editions as a touchpoint around which to coordinate efforts. For the 2018 edition this meant a lot of personal sacrifices and individual heroism, to the point that for 2021 we decided to scale back our aim and instead approach the effort with a more reasonable workload for the people involved. In the project we are experimenting with new ways of organizing ourselves, like for example the [Async Foundation Working Group's Vision document](https://blog.rust-lang.org/2021/04/14/async-vision-doc-shiny-future.html).

But, in my eyes, our ecosystem cannot operate in the way it has so far indefinitely. Increased adoption comes along with increased responsibilities to our users. More contributors require more coordination and "administrative" work to make sure all of us share a common vision. And we need visibility into how the many repositories and initiatives that make up "Rust".

What makes Rust "Rust" isn't "just" the compiler, or the official tooling: Rust is an ecosystem of tooling, documentation, libraries and *people*. Many of these are *independent* of the official project and also maintained by a handful of people in their spare time.

The long term health of the Rust ecosystem is very important, and ensuring it requires for us not only to grow over time, but to make sure we can scale appropriately. And accomplishing that means making sure that critical parts of our ecosystem are "well staffed" and capable of maintaining themselves and that no individual maintainer burns out.

<a style="float: right; margin: 2em;" href="https://xkcd.com/2347/"><img src="//imgs.xkcd.com/comics/dependency.png" title="Someday ImageMagick will finally break for good and we'll have a long period of scrambling as we try to reassemble civilization from the rubble." alt="Dependency" srcset="//imgs.xkcd.com/comics/dependency_2x.png 2x" style="image-orientation:none"></a>

Newcomers interested in helping are driven, but the community could do better to help them find where they can do so. Individual teams have preasence in Zulip where we can individually help them. It has documentation for new contributors. The backlog is well groomed and all of the teams operate in the open. Projects from the wider community operate in different ways, but usually follow similar arrangements.

But a desire to help, capacity and time, left undirected will not have the full impact it could have. Knowing *where* to direct people is almost as important as having people approaching us and asking. But for this we need visibility: we need to know what the health of the ecosystem is, and where we need to focus our efforts at.

I can currently look at the activity of individual projects, by going to their repositories, backlogs and public conversations and talk with people, but this is time consuming and error prone. In order to have ongoing visibility into the health of the ecosystem, I believe having more automated sources of information are needed. But these data points *should not* be the start and end of the story, they should be merely another tool in the toolbox, a way to have early signals that we need to talk to people to find out more.

Anyone that has ever released a new service knows about the importance of The Dashboard. Before putting a project in front of its users, every person tasked with making sure that it continues to work correctly for all of them will go through the excercise of identifying what the "steady state" of the system is, and instrument their code to collect this data in a centralized place. This allows the service owners to find out when things go wrong *before* (or at worst, at the same time as) their users. Releasing a *new* service is always "exciting": the behavior of the system at load is different than what it was during testing, what was expected to be uncommon scenarios might happen much more often than expected, and problems that were *not* anticipated might arise in such a way that are "invisible" to the dashboard you've built.

The same principles can apply to operating with groups of people. Having aggregated information about the state of a project is useful in the same way as it is for a service. Crucially, people *are not computers*. What can be unnacceptable levels of commitment that will burn them out quickly for one person, can be how others thrive. Different codebases have different needs for what it means to "be in a healthy state". But having more information, hard numbers to lead us to ask more targetted questions will be useful in getting *ahead* of people suffering burn-out, help us identify problems (and things that are going well!) earlier that we otherwise would.


This is why I find the ongoing efforts around [Optopodi] incredibly important and I want to expand on them. Optopodi currently is a CLI tool that queries a list of GitHub repositories for data. We can use it to identify projects that:

* Are struggling with their issue backlog
* Having a decrease in their numbers of contributors
* How many people are regularly doing code reviews

There are specific questions I would love to able to answer, like:

* What parts of the compiler are changing and how often?
* How long to reviews normally take?
* How often do people *stop* contributing?
    * Can we understand *why* they did?
* and more questions that I haven't even *begun* to realize we need to answer

My long term objective is for this tool to become a full-blown automated service feeding a public dashboard: I want to be able to check what the state of the ecosystem is at a glance and for everyone to be able to see the same data I am looking at. I want to be able to know when contributors are at the risk of burn out and prevent it. I want to ensure that the "Quality of Service" for contributions is high, and if it isn't address it and be able to verify that our changes have the intended effect. I want us to have clear visibility into the effect our actions have. I want us not to wonder whether we are doing the right thing, I want us to have concrete proof.

Having this information available, not only will let the project direct our efforts, it lets us do so *early*, by being able to detect potential issues *before* they are *felt* by the maintainers. Increasing the number of knowledgeable contributors involved in critical libraries helps *every* contributor in those projects: PRs and issues will have faster turn-around. And having this information out in the public allows others to independently evaluate our health as well.

If the Optopodi tool and its goals are appealing to you, you can [see the current code base on GitHub](https://github.com/optopodi/optopodi), where there are [outstanding feature requests you can implement and where you can file new tickets](https://github.com/optopodi/optopodi/issues) to shape the tool, and as well as reach out to it's contributors at the [GitHub Metrics Zulip Chat](https://github-metrics.zulipchat.com/). And if you have thoughts on this blog post and the vision presented, independently of the tools, I am also interested in hearing from you on [Zulip](https://rust-lang.zulipchat.com/#narrow/pm-with/119031-user119031) and [Twitter](https://twitter.com/ekuber/status/1453078598160506880).


[Optopodi]: https://github.com/optopodi/optopodi
