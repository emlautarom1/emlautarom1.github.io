---
title: "A Haskell effect system"
date: 2025-03-03T22:04:51-03:00
summary: What does it take to build a minimal effect system?
draft: true
---

## Introduction

For quite some time I've been interested in effect systems for Haskell, in particular in those systems that allow developers to build programs where interfaces and implementations are fully decouple: programs should list the "operations" they're interested in performing (the "what") while the implementations of these operations are defined somewhere else (the "how").

In recent years, effect systems have gained a lot of popularity in functional programming languages as a way to structure programs in this fashion (TODO: add Polysemy video) yet implementations of these systems are usually quite intricate while also compromise in boilerplate, performance and vendor lock-in: it's extremely hard for junior developers to understand how or why these effect systems work, they require some level of code generation that usually is handled via `TemplateHaskell`, and can negatively impact performance in unexpected ways.

## What are we doing

With this in mind, I decided to try and implement an effect system using the most minimal set of features possible while getting a result that works **for me**. The later clarification is quite important: I would like to build something that works for the use cases that I have in mind without generalization as a goal.

## State of the art

The current effect system landscape has settled mostly in the `ReaderT IO` approach

## Implementation

## Limitations & downsides

## Conclusions
