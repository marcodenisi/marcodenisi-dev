---
title: "Why you should use Conventional Commits"
date: 2020-01-10T10:30:41+01:00
draft: false
categories: ["Development"]
tags: ["git", "semantic-versioning", "changelog", "automation", "cd"]
---

Nowadays, versioning systems are used everywhere to simplify collaboration between developers working on the same codebase. That's great, if it weren't that, in the long run, commits history starts looking like this:

![Git Commit - xkcd](/img/articles/git_commit.png)

Has a commit introduced a breaking change? Any fix? Good luck finding the answer just looking at that: you'd better start walking in your open space asking who did what.

From this another problem arises. Let's say you have to decide the software version number. Usually, it is composed by three numbers `X.Y.Z`. How do you decide these values? And how do you create the changelog?

Let's start by talking about **Conventional Commits**.

# What's Conventional Commits

Conventional Commits is nothing but a *convention* on how commit messages should be written. As stated by the [official documentation](https://www.conventionalcommits.org/en/v1.0.0/), commits should be in the form:
```
<type>[optional scope]: <description>
[optional body]
[optional footer(s)]
```

And here a practical example, without optional fields:
```
feat: add new API to retrieve users
```
Main types could be:

- `feat`, describing the introduction of a new feature
- `fix`, describing a fix
There are a bunch of them, you can find more information in the documentation.

# Why it's useful

Sometime a *git log* is worth a thousand words:
```bash
fix(lambda-article): change publish date
fix(lambda-article): make article not draft anymore
feat(lambda-article): add new article on lambda functions
feat(cd): introduce github action to automatically build website
```
Just reading commit descriptions you can exactly understand what has been done. In this example, two features and two fixes were introduced.

You may say that you can do the same without using conventional commits. You're right, so let's talk about **Semantic Versioning**.

# Semantic Versioning

[Semantic Versioning](https://semver.org/) it's a set of rules (or a *framework*) defining how to decide the version number.

Given a version `X.Y.Z`:

- X is the MAJOR: when introducing *breaking changes*
- Y is the MINOR: for new *features*
- Z is the PATCH: for new *bug fixes*

The link with Conventional Commits is obvious: from commits history you can exactly define the version number to assign.

Lot of tools can also autogenerate changelogs starting from commits history: here at Sky we use an internal one (*what-bump*, written in Rust), but there are a lot of open-source alternatives.

Just add these kind of tools in your release pipeline *et voilà*, version number set and changelog generated.