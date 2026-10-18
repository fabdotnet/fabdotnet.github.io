+++
title = "Why I Use AI to Build My Tools"
date = 2026-10-18T00:00:00Z
lastmod = 2026-10-18T00:00:00Z
draft = false
tags = ["ai", "tooling", "meta"]
description = "How I use AI to ship my security tools faster, and what remains mine: research, design, testing and validation."
summary = "How I use AI to ship my security tools faster, and what remains mine: research, design, testing and validation."
+++

## Ideas Were Never the Bottleneck, Time Was

I have always had ideas for tools: small utilities that solve a problem I keep
running into, and that other people in the community would probably find
useful too. I have always wanted to share them.

The hard part was never the idea. Turning it into something worth publishing
takes real development time, and even more testing time. Between work and
everything else, most of my ideas stayed ideas, or ended up as a half-finished
script on a disk somewhere.

AI changed that. Development time has dropped so much that some days I wonder
whether finding the right idea now takes longer than building it. ^^

## What I Do vs. What AI Does

**What stays mine:**

- spotting the need and coming up with the idea;
- the design and the architecture choices;
- vulnerability research, analysis and coordinated disclosure: the CVEs
  published so far were found by me, without AI;
- testing and validation;
- the decision to publish.

**What AI speeds up:**

- writing the code of the tools published in [Research & Tools](/tools/);
- writing their documentation;
- drafting and formatting the articles on this blog, CVE write-ups included.

## How I Keep It Trustworthy

A buggy tool does more harm than no tool at all, so nothing gets released
without being tested first: in a lab, under real conditions, on several
operating systems when the tool targets more than one.

That does not mean every possible case has been covered. If I publish
something, it is because I have high confidence in it, not perfect
confidence. These projects are built on my free time, on a best-effort basis,
so some bugs may still slip through ;)

Everything published here is reviewed, tested and owned by me. Using AI is a
tooling choice, like using a framework or a debugger: the responsibility for
what ships stays with me.

## Feedback

If something looks wrong in a tool or in an article, open an issue or a pull
request on [GitHub](https://github.com/fabdotnet). That is the fastest way to
get it fixed.
