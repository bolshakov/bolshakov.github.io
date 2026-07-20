---
layout: post
title: "Can a Candy Bar Price a Code Review?"
date: 2026-07-20 00:00:00 +0000
---

In the mid-1990s, ten daycare centers in Haifa had a problem: parents showed up late to pick up
their kids. A teacher had to stay late every time, unpaid, waiting. So the daycares ran an experiment.
Six of them introduced a small fine for picking up more than ten minutes late.

Late pickups didn't go down. They roughly doubled, and stayed doubled even after the fine was later
removed.

## Why it backfired

Uri Gneezy and Aldo Rustichini, the economists who ran the study, had the following explanation: before the
fine, showing up late meant a teacher was doing you a favor. That's a social obligation, and guilt is a
strong incentive. After the fine, showing up late meant paying a few dollars for an extra twenty minutes
of babysitting. That's a monetary transaction, and a cheap one. The fine didn't put a price on lateness
-- it announced that lateness was for sale, and named the price.

Removing the fine afterward didn't restore the guilt. Once a relationship has been reframed as monetary,
you can't reframe it back by changing the price to zero. It's one of those findings that ruins a whole
category of intuitive fixes once you've seen it: **you cannot assume that pricing an undesirable
behavior discourages it.** Sometimes it just legitimizes it.

## Code review has the same shape

Most code review still runs on the pre-fine mechanism. Nobody invoices you for reviewing a coworker's
PR. You do it because you'd want the same back, because norms say so, because leaving someone's PR
unreviewed for three days feels bad. It's a social obligation, held together by reciprocity and mild
guilt -- exactly like the daycare pickup, before anyone touched it with a price.

And that mechanism is changing. AI-assisted coding didn't make review easier -- it made there be far
more of it. A [Faros AI telemetry study](https://www.faros.ai/blog/ai-software-engineering) from July
2025 tracked more than 10,000 developers across 1,255 teams. The ones that relied hardest on AI, merged
98% more pull requests, and bigger ones -- average PR size up 154%, review time up 91% -- with no
matching gain in whether any of it actually shipped faster. That's a correlation across teams, not proof
that AI caused it. But the shape is hard to miss: the bottleneck moved from writing code to reading it,
and nothing in the workflow says so. A 600-line PR and a 60-line PR show up in the queue looking
identical -- one line in a list, no cost attached, no signal that one of them is going to eat someone's
afternoon.

The obvious fix is to supply the missing signal: estimate what each PR will cost to review, in minutes,
and post that number right on the PR. Which should make you nervous, because that's what the
daycare did. But the daycare did two things at once: it named a price *and* it collected the money, and
the study can't tell you which half did the damage. That number only does the first half -- it says what
a review will cost and collects nothing. So maybe the danger doesn't transfer: the sharp end of the fine
was the invoice, and there's no invoice here.

Someone pulled those two halves apart, though, and the result isn't reassuring. Not with daycare
pickups -- with candy. Heyman and Ariely had people do a dull little task -- dragging shapes across a
screen -- in exchange for a gift, and told one group exactly what the gift was worth: "this retails for
50 cents," with no money changing hands. That group stopped working like they'd been handed a favor and
started working like they'd been paid -- effort dropping when the gift was cheap, rising when it was
pricey, the shape of an actual wage. Naming a price, with nothing payable, was enough by itself to flip a
favor into a transaction.

Which leaves us with a question: does a bot commenting "this PR costs about 40 minutes of someone's
attention" do to a code review what naming the candy's price did to a favor -- flip it into a
transaction?

I don't know -- but I don't think the candy study gets the last word here, because a pull request isn't a
wrapped gift. And notice who flipped in that experiment: the person doing the work, which in a code review
is the reviewer. The number doesn't land on the reviewer first, though. It sits on the PR, so the person
who sees it before anyone else is the one who wrote it -- and the two of them read it completely
differently. Take them one at a time.

Start with the developer, the one being nudged to split. The candy priced *your own* effort. This number
prices what you're about to ask of *someone else* -- which is exactly where the reciprocity and mild
guilt that ran code review in the first place still live. A 50-cent gift labeled "50 cents" just feels
cheap. But a PR labeled "two hours of a colleague's afternoon" might not read as *now it's a
transaction*. It might read as *that's a big thing to ask* -- the same guilt code review always ran on,
only now with a number making it concrete. So the very same label could cut two ways: it could let the
developer off the hook -- *I paid the toll, we're square* -- or press the obligation harder. Which way it
lands is the whole question.

The reviewer never faces that question. To them the number isn't a price at all, it's an anchor. "About 40
minutes" sets an expectation of how much attention the thing deserves, and people drift toward the number
they're handed -- which cuts both ways. It can pull a five-minute skim up into a real review, or drag a
careful hour-long reader down to forty. Whether that helps depends entirely on whether the estimate is
any good -- and it's mostly line count, a rough proxy: a 40-line concurrency or regex change can burn
more of an afternoon than 300 lines of boilerplate.

So there's more than one mechanism here, and they don't point the same way. I built the tool betting the
guilt sharpens the obligation rather than dissolving it, and that the anchor helps more than it hurts
-- but those are bets, not findings.

## What that looks like as a tool

[ProQuo](https://github.com/bolshakov/proquo) is that bet, shipped as a GitHub Action: it looks
at a pull request's diff, computes an effective size (lockfiles and generated code excluded, tests
and comments down-weighted -- they're real, but they read faster), and posts a comment estimating the
focused review time in minutes. On a large PR it reads about like this:

> 🔴 **488 effective lines** — about **59–146** min of focused review, more than comfortably fits in a
> single review session. Beyond 400 lines, review effectiveness drops and reviewers tend to
> leave fewer useful comments per line.
>
> **Worth splitting?** Breaking this into ≤200-line PRs puts each piece back in the size band where
> reviewers catch the most defects per line. The split buys detection quality, not saved review minutes.

That comment only lands once you open the PR -- but the problem I started with lived in the queue, where
the 600-line PR and the 60-line one look identical. So ProQuo also tags each PR with a size label,
dropping the signal into the list itself, before anyone clicks in.

![Pull request list, each row tagged with a ProQuo size label](/assets/can-a-candy-bar-price-a-code-review/proquo-size-labels.png)

The size bands and the rate come from a SmartBear/Cisco field study of 2,500 reviews: somewhere past 200
to 400 changed lines, reviewers start catching fewer defects *per line*. Read that carefully, though --
"defects per line" is a ratio, and it can shrink just because the line count grew, not because the review
got any worse. The effect is real. It's just softer than that number first makes it sound.

It's a label, not an invoice -- it doesn't add a cost, it reveals one. The reviewer's afternoon was
always the price. And a cost you can see, one you're about to hand a real person, is harder to shrug off
than an invented one -- which tips the bet toward guilt sharpening, not dissolving. Push a giant diff and
nobody stops you -- you just see the number while you can still act on it.

Mostly I'm curious whether the number moves anything at all -- whether seeing it before you open the PR
nudges you to split, or whether naming the cost quietly turns the favor into a
transaction, the way the fine and the candy did. Trying it on a repo is a few lines of YAML: [the README has the steps](https://github.com/bolshakov/proquo#usage).

I've got no way to measure this from the outside, so I'm just going to ask. If you install it and
something moves, good or bad, [tell me about it](https://github.com/bolshakov/proquo/discussions).
