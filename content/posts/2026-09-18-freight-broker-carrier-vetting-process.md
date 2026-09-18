---
title: "freight broker carrier vetting process"
date: 2026-09-18
draft: false
tags: ["carrier vetting","double brokering","freight broker","carrier fraud","onboarding"]
categories: ["Freight & Logistics"]
description: "A practical carrier vetting process that actually screens out double-brokers and fraudulent carriers, built by people who move freight for a living."
showToc: true
---

Every broker we talk to has a story. A load tenders clean, the carrier confirms, the truck shows up on the BOL, and then the freight vanishes into a re-broker chain three layers deep. By the time anyone notices, the shipper is calling about a missed delivery and the "carrier" on the RateCon isn't the outfit that actually hauled it. Double brokering and identity theft in this industry aren't rare edge cases anymore — they're a cost of doing business unless your vetting process is built to catch them before the truck ever gets dispatched.

The problem isn't that brokers don't care about vetting. It's that most vetting happens once, at onboarding, and then never again. A carrier can pass every check on day one and still hand your freight to a stranger on day two hundred. If your process stops at "checked FMCSA, looked fine," you're not vetting — you're checking a box.

## Why a One-Time Check Doesn't Work

FMCSA's SAFER system and the Licensing & Insurance portal will tell you a carrier is active, has authority, and carries insurance on file. That's necessary. It's not sufficient. None of it tells you whether the MC number you're tendering to is the same outfit that shows up at the shipper's dock. Authority can be legitimate and still get rented out, borrowed, or spoofed.

The fraud patterns that actually hurt brokers running 10-75 trucks worth of freight a week tend to fall into a few buckets:

- A carrier with real authority accepts the load, then re-brokers it to a second unvetted carrier without telling you — classic double brokering.
- Someone impersonates a legitimate MC number using stolen letterhead and insurance certs, quotes low, and disappears with the freight.
- A carrier's insurance lapses mid-relationship and nobody catches it until there's a claim.
- Contact info on file doesn't match the driver who actually shows up at pickup.

None of these get caught by a single lookup at signup. They get caught by a process that treats vetting as ongoing, not a one-time gate.

### What Actually Screens These Out

Start with the obvious layer, because skipping it is how people get burned in the first place. Pull FMCSA authority status, insurance filings, and safety rating every time a new carrier comes on, and re-pull before any high-value tender if the carrier hasn't moved freight for you recently. A carrier that was active and insured ninety days ago isn't automatically active and insured today.

Then go past the government data. Call the carrier back on the phone number listed in FMCSA's registration, not the number on the quote email or the signature block. If those two numbers don't match, that's not automatically fraud, but it's a reason to ask more questions before you tender anything. Confirm the MC number, the DOT number, and the company name all line up — mismatches are one of the cheapest tells in a double-broker scheme.

Ask for a copy of the carrier's own certificate of insurance directly from their agent, not forwarded through a broker or freight-matching contact. Verify it independently with the insurance company. This single step stops a huge share of the "borrowed authority" schemes, because the fraud usually can't survive a direct call to the actual insurer.

Watch how a carrier behaves once freight is moving, not just how they look on paper. A truck that goes dark between check calls, a driver who won't answer a direct call, a POD that comes back from a different company name than the one on the RateCon — these are operational signals, and they show up in your dispatch and tracking data before they show up in a compliance report. This is where the vetting process stops being a one-time form and becomes something your team can actually see happening in real time. We built the carriers module inside Techvia TMS because the dispatchers we talk to needed a place to track authority status, insurance expiration, and carrier performance history side by side with the loads those carriers are actually running — not in a separate spreadsheet that nobody updates after week one.

## Building the Habit, Not Just the Checklist

A vetting process only works if somebody owns it and it runs on a schedule. That means:

Set a recurring date to re-verify authority and insurance for every active carrier, not just new ones. Insurance certificates expire; authority gets revoked; none of that shows up unless someone looks.

Log every check call, every POD mismatch, every late pickup against the carrier's record, not just the load's record. A carrier who's flaky on three loads in a row is a pattern, but only if someone's tracking it across loads instead of load by load.

Require confirmation of the actual driver and truck before dispatch on any new carrier relationship, and cross-check that against what shows up at pickup. A name and phone number that don't match the paperwork is worth a phone call before the truck leaves the yard.

Keep a short list of red flags your team checks without having to think about it: MC numbers younger than six months paired with fleets that seem too large for that age, rates that are dramatically below market, insurance certs that arrive from an email address unrelated to the carrier's registered domain, and carriers who push hard to skip a direct verification call.

## What This Costs You If You Skip It

The math on skipping vetting isn't abstract. If the freight gets double-brokered and lost, you're paying the shipper's claim out of your own margin, and you're doing it on a load where you already thought you'd covered your risk. If a carrier's insurance lapsed and there's an accident mid-haul, you may be fighting a cargo claim with no coverage behind it. And every one of those events costs more than a phone call to the insurance agent would have.

The brokers and carriers running 10-75 trucks who handle this well aren't doing anything exotic. They're running the same three or four checks every time, logging what they see, and treating carrier history as data they can pull up in seconds rather than something they have to reconstruct from memory after something's already gone wrong.

## Where To Go From Here

Vetting isn't a form you fill out once and file away — it's a habit that has to survive contact with a busy dispatch board and a full lane sheet. If your current process is a folder of PDFs and a hope that nobody's lying, it's worth looking at how a system built for carriers, not just loads, changes what your team catches before the truck leaves the yard. Take a look at how the carriers module works inside [Techvia TMS](https://www.techvia.software/products/tms) and see if it fits how your team already operates.