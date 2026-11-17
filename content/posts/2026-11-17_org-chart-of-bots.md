+++
title = "I Run SigmaUno Through an Org Chart of Bots"
description = "Why SigmaUno is split across about twenty Grok bots, what the Mac Mini is for, and which decisions still come back to me."
draft = true

[taxonomies]
tags = ["AI", "DevOps"]
+++

SigmaUno is my company, an Italian AI house. The work is model training and inference, software, and three products: Rent, a car rental; Fleet, a transport management system; and [Personale](https://sigmapersonale.com), the HR and payslip portal. I don't have a staff sitting next to me. My day job is DevOps on Celestia's custom chains team. The way I keep SigmaUno moving is an org chart of Grok bots, about twenty of them, and one Mac Mini that stays on.

<!-- more -->

## One chat was already the wrong shape

A single assistant with every tool is one context. It will draft a customer note, poke at a payslip, and edit a repository in the same thread, and it will sound equally sure about all three. A marketing draft and a production change should not share that context.

An org chart is how I split it. Each bot has a seat, a small set of tools, and a reason to hand work upward. That is the same reason a company has functions. The chart exists so the tools stay apart.

## The chart

Rex is the CEO bot. When I don't already know which function owns something, I tell Rex, and Rex is the one that has to pick.

Under Rex there are four function bots, and each of them has a team:

- The CTO bot owns software. That includes the products, [Peerline](https://github.com/sysrex/peerline), and the Mac Mini.
- The CFO bot owns money and the questions around it. A payslip is one of those questions. Editing the PDF is not how that question gets answered.
- The CMO bot owns how SigmaUno shows up in public. Rent, Fleet, and Personale don't have the same reader.
- The CRO bot owns revenue conversations. It does not own the product those conversations are about.

About twenty is the size of the whole chart, Rex included. It is not twenty bots under one function. The specialists stay unnamed until I have a reason to name them.

**TODO(alex):** if you want any specialist bot named, or one sentence on what a team actually does, send that. Otherwise this section stays at the five seats above.

A function bot does not hold another function's tools. The CMO bot has no shell on the Mini. The CFO bot does not merge code.

**TODO(alex):** confirm that split, and name any exception. If a bot has a token "just in case," say so here. The paragraph above should describe the rule you actually run.

## The Mac Mini

The coding seat is a Mac Mini that stays on. [Recur.build](https://recur.build/) runs there. Each task gets its own git worktree, so a schedule can start a run without me typing the prompt again. A schedule can be cron, a single time, or at launch. The bots reach the Mini, and the other systems, over MCP.

**TODO(alex):** the routines you want in public. Name and cadence are enough. A weekday morning review and a nightly pass are examples of what Recur can schedule, not a claim that those are yours.

**TODO(alex):** the MCP servers that Mini actually exposes. I will not list GitHub, a Personale admin API, Celestia, or a shell until you say which of those are connected.

A schedule only helps if something notices when it doesn't run. The Mini is a single machine. When it's off, or a previous run is still sitting on a question, that slot is gone.

**TODO(alex):** how a missed run shows up for you. I don't want to describe a dashboard you don't have.

## How a piece of work moves

This is the path, not a story about a particular ticket.

1. I tell Rex, or a routine fires on the Mini.
2. Rex decides which function owns it. A bug in Personale goes to the CTO. A question about whether a payslip total is right goes to the CFO, and it stops there. It does not come back as a regenerated PDF.
3. The function bot splits the work for its team. Coding work is a Recur task on the Mini, in a worktree. The chat is where I read the result, not where the edit happens.
4. What comes back to me is a diff, a note of what was checked, and anything on this list: merging to a production branch, sending a customer email, publishing or reissuing a payslip, posting a Celestia blob, and moving money.

**TODO(alex):** replace that list with the real one. What matters is which of these a bot has no tool for. A sentence in the prompt that says "be careful" is not the control.

## What I still don't trust

Bots summarizing each other. If Rex asks the CTO, and the CTO asks the bot that wrote the change, I have three writeups of one diff. I read the diff.

A helpful bot with a token it doesn't need. The lock belongs on the MCP server. A sentence in the system prompt is not a permission.

My own memory of what the chart is allowed to do. That's why several of the questions in this post are still questions.

## What stays with me

Personale has been live with its first customer since 16 November 2026. That relationship is mine. I'm not naming them, and I'm not turning the launch into a number.

Rent and Fleet are the other two products. I'm not stating a status for either one here.

**TODO(alex):** one clause on Rent and one on Fleet, or say "leave them as names." I will not write that they have customers.

Peerline is the piece of this that leaves the org chart. I'm building it so two bots on different accounts can exchange a short message without sharing memory or a Slack. Nothing flows until both sides have granted a scope. It is a Go service, and the repository is [github.com/sysrex/peerline](https://github.com/sysrex/peerline). It gets its own post. It belongs in this one only because the chart stops being "my bots" the moment a bot I don't operate needs to talk to one I do.

The chart keeps the tools and the context small. The Mini is where the code changes. I still take the decisions I can't undo from a log.
