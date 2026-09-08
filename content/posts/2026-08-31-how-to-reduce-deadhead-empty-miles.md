---
title: "How to Reduce Deadhead and Empty Miles"
date: 2026-08-31
draft: false
tags: ["deadhead miles","empty miles","dispatch efficiency","fleet management","freight margin"]
categories: ["Dispatch"]
description: "A practical method for calculating what deadhead actually costs your fleet, plus the dispatch levers carriers and brokers use to cut empty miles."
showToc: true
---

Every dispatcher who has run a board for more than a few months has a number in their head for deadhead. Most of them are wrong. Not because they're bad at math, but because nobody sat down and worked it out from actual settlement data — they estimated it once, three years ago, and it stuck.

We built Techvia TMS because the dispatchers we talk to were making load acceptance calls off gut feel and a mental average, when the real number was sitting in their own RateCons and driver settlements the whole time. Deadhead isn't a mystery. It's arithmetic you're avoiding because pulling the data by hand is a pain.

## What Deadhead Actually Costs You

Start with the definition that matters for your P&L, not the textbook one. Deadhead is any mile your truck runs without a paying load on it — repositioning after a delivery, running to a shipper for pickup, or backhauling empty because nothing penciled. It's not wasted time in the legal sense; the driver is still on the clock, still burning fuel, still accruing wear on tires and brakes. It's just mileage with no revenue attached.

To find your real cost per deadhead mile, you need four numbers you already have:

- Your all-in cost per mile (fuel, driver pay, insurance, maintenance reserve, ELD and permit costs divided across annual miles)
- Total miles run last month, loaded and empty, pulled from your ELD or trip logs
- Total loaded miles from your dispatch records or settlement history
- Total revenue booked for that same period

Subtract loaded miles from total miles to get empty miles. Divide empty miles by total miles to get your deadhead percentage. Then multiply your deadhead miles by your all-in cost per mile — that's what you spent moving nothing. Do this per truck, not just fleet-wide, because a 15% fleet average can hide one lane pair running at 30% empty and dragging the whole number down.

Do this exercise every month for a quarter and you'll start to see which lanes, which customers, and which dispatchers are quietly bleeding miles. That's the whole point — not a one-time audit, but a habit.

### Why Fleet-Wide Averages Lie to You

A 40-truck fleet with a 12% average deadhead rate sounds fine until you break it down by terminal or by dispatcher and find that six trucks running a specific regional lane are sitting at 25%+ empty because the backhaul market on that corridor is thin and nobody's adjusted the lane assignments in a year. Averages are useful for board-level tracking. They're useless for fixing anything. Pull deadhead by truck, by lane, and by dispatcher if you run more than one, and the fix usually becomes obvious fast.

## The Dispatch-Level Levers That Actually Move the Number

Once you know where the empty miles are coming from, there are a handful of concrete levers that work — no software required to try them, though software makes them faster.

**Tighten your radius before you tender acceptance.** A lot of deadhead gets baked in at the moment a dispatcher accepts a load without checking what's on the other end. If a load delivers into a market with thin outbound freight, you're pricing in the empty return before the truck even leaves. Build a habit of checking historical outbound volume for the delivery zip before confirming, not after.

**Stack multi-stop loads instead of single-pickup runs where the lane supports it.** Every additional stop you can consolidate onto one truck reduces the number of separate deadhead legs you'd otherwise run to string together two single loads. This matters more for regional and dedicated lanes than for long-haul, but it's underused across the board.

**Negotiate backhaul rates into your RateCon at time of booking**, not as an afterthought after the truck's already empty in the destination market. Brokers who know a lane runs thin on the return leg can build that into the linehaul rate up front instead of asking the carrier to eat it. Carriers should be asking for this explicitly — it's a normal conversation, not a favor.

**Reroute drivers proactively based on live position, not last known check call.** If you're still relying on phone check calls to know where a truck sits, you're making backhaul decisions on stale information. A driver who was two hours from a hot market when he checked in at 8am might be sitting in a different market by noon. Live GPS visibility — whether from ELD integration or a tracking link the driver opens on his phone — closes that gap and lets dispatch react while there's still time to book something instead of running empty.

**Rank load options by margin after deadhead, not gross rate.** This is the one most fleets skip entirely. A $2.40/mile load that requires 180 empty miles to reach can net worse than a $2.10/mile load sitting 20 miles from where the truck just dropped. Gross rate tells you almost nothing on its own. This is exactly the gap our AI Dispatcher in Techvia TMS was built to close — it ranks available loads by projected margin after deadhead automatically, so dispatchers aren't doing that math by hand under time pressure while three other trucks are also waiting on assignments.

## Build the Habit, Not Just the One-Time Fix

None of these levers work as a single cleanup pass. Deadhead creeps back in the moment your dispatch team goes back to booking on gross rate and geographic instinct instead of margin math. The carriers who keep their empty-mile percentage down over time treat it like fuel cost or on-time percentage — a number they check weekly, broken down by truck, and act on immediately rather than reviewing in a quarterly meeting after the damage is done.

Pull your deadhead number this month using your own settlement and mileage data. Do it by truck. Then decide which lever above actually applies to your freight mix — thin-market backhaul negotiation looks different for a broker sourcing capacity than it does for an asset carrier dispatching its own trucks.

If you're doing this math by hand right now, pulling numbers out of three different spreadsheets and a stack of PODs, take a look at how Techvia TMS handles it — drag-and-drop dispatch, live GPS tracking, and margin-and-deadhead load ranking built into the board itself, flat $199 a month for the whole team, no per-seat pricing. You can try it for 30 days with no credit card at [techvia.software/products/tms](https://www.techvia.software/products/tms).