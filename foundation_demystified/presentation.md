# Rust Foundation: Demystified

---

# What we do

- Support the project

^ So for starters I want to talk about what the foundation actually does. Fundamentally, we exist to support the project. This comes in a couple of different forms

---
[.build-lists: true]

# Supporting the project

- Providing grants
- Hiring full time engineers
- Paying infrastructure costs
- Providing access to lawyers

^ At its core, the biggest thing the foundation does is raise funds from various companies who are using Rust and funnel it into the ecosystem. This is most directly done through our grants program, where folks inside the project and the community submit proposals for things they'd like to work on, and they receive some amount of funding from the foundation to work on it. But depending on where in the world the grantee lives, this usually isn't enough to cover all their costs. This is why when we're able we also hire full time engineers whose only job is to work on the Rust project. This is something we generally do sparingly, as we don't want to be in a situation where only a handful of people benefit from the foundation. We also need to make sure we're in a situation where we can give our engineers reasonable job security. We never want to be in a position where the foundation is having to do layoffs.

^ But engineering costs are only one part of what it takes to run the Rust project. Rust has a lot of infrastructure that its built on top of, and this gets *very* expensive. If none of Rust's infrastructure was donated, it would cost tens or even hundreds of thousands of dollars per month to pay for Rust's CI, file hosting, tools like Crater which tests a change to the compiler by compiling every crate on crates.io, and other things. (this overlaps with JD's talk, cut it and plug that talk?)

^ When the project needs something that costs money, the Foundation will either get that donated by one of our members, or pay the cost from our budget otherwise.

^ And finally, the project has more legal needs than you might expect. Crates.io has to deal with copyright law (whether that's in the form of DMCA takedown requests or other forms). And the project wants to continue to hold the trademark for Rust. What needs to happen to ensure we keep it? There's also random situations that can come up like: "Rust has a lot of US contribuors. Do we need to care about the US sanctions on Iran when reviewing pull requests?". Also in most countries, people can just sue you for nearly any reason. What if you get sued for work you did on behalf of the project? All of these situations require legal resources, which the Foundation provides.

---

# Who is the Rust Project?

^ needs to be said somewhere, but doesn't really fit anywhere

---

# What we do

- Support the project

^ Alright so we've talked about the ways in which we support the project, but what else does the foundation do?

---

# What we do

- Support the project
- Be a legal entity

^ The next role that the foundation fills is acting as a legal entity for "Rust", when it's needed. When the foundation was originally formed, the MVP was "a legal entity that can hold the trademark that isn't Mozilla". But the need for a legal entity encompasses more than just the trademark, and I'd like to tell you about a case where this happened during my time as the crates.io team lead. This happened years ago, long before the foundation existed. Github token scanning was in an early preview stage. At this point there wasn't easy way for projects to join that system. I had quite a few contacts over there, so I wanted to see if there was some way to get crates.io added in this early stage. Since this was in such an early preview, the first thing that needed to happen was to sign a bunch of legal documents.

---

# Can Rust sign an NDA?

^ Now just to be clear, this isn't meant to rag on GitHub at all. Given that at the time this feature was focused on companies like AWS, and that this feature was *super* early access, it's completely reasonable for them to have expectations that were meant for companies. But I wasn't a company, I was a single open source contributor.

---

# [anakin meme]

^ So my reaction was like "uh....... Can I sign this contract? I can definitely sign this stuff for myself but I don't think I have the authority to sign this on behalf of everyone else in the Rust project". But that wasn't something that was possible at the time. I spent some time talking to folks within Mozilla legal to see if it was possible to have them act as that authority, but it wasn't something they were willing to do. So this just didn't end up happening. GitHub recommended setting something up in OpenCollective, but I didn't feel like I had the clout to take the lead on that.

---

## (GitHub token scanning supports crates.io now)

^ And just to reiterate, this isn't meant to rag on GitHub *at all*. As the feature left preview status, bureaucracy like this went away. And eventually the crates.io team did get integrated. I don't think anything about this was wrong on GitHub's part, the project just wasn't set up to engage with corporations like that at the time.

---

# Corporate Bureaucracy

^ And while this case worked out in the end, there's plenty of cases where this sort of bureaucracy never goes away. At the end of the day, *any* entity signing something like an NDA on behalf of an open source project is going to be pretty pointless. But saying that the bureaucracy shouldn't exist doesn't make it so. And this isn't something that folks come to open source to deal with. Which is why we need a foundation to deal with it for you.

---

# Navigating Corporate Structures

^ And this comes up in more cases than just signing legal documents. Dealing with giant corporations as an individual contributor can be incredibly overwhelming. In the years prior to the foundation forming, I was working on crates.io full time -- which meant I needed to find funding sources in order to pay my bills. It was pretty easy to get some support from Mozilla, since they actually had a team dedicated to Rust which had a manager and a budget. But I wasn't expecting them to cover all my costs forever. I needed to get support from other companies. There was a lot of chatter at the time that "Google wants to give more to Rust", "Microsoft wants to give more to Rust". Which is great! But how do I, as an individual contributing to Rust and wanting funding, get connected to the person within that company who can actually sign a contract? Navigating those structures was extremely time consuming. And I wasn't very good at it. I was a programmer, I wanted to spend my time programming. But I was quickly finding myself spending somewhere between 1/3 to 1/2 of my time just trying to make sure I could pay my bills. At which point it was just easier to take a part time job than try to deal with it.

---

talk about problems of full-time contributors working for companies here?

transition into how companies contribute to the foundation and then to how the board works feels natural
