---
title: "How to Stop Running Dispatch on Spreadsheets"
date: 2026-08-24
draft: false
tags: ["dispatch software", "spreadsheets", "TMS migration", "carrier operations"]
categories: ["Dispatch"]
description: "A week-by-week plan for moving dispatch off spreadsheets and onto a TMS, including the parts that break and how to handle them."
showToc: true
---

Every carrier and brokerage we talk to started the same way: a spreadsheet someone built on a slow Tuesday, and it never stopped growing. Now it's got fifteen tabs, three macros nobody remembers writing, and a driver pay formula that only works if you don't sort column G. It's not that spreadsheets are bad. It's that they were never built to run dispatch for a fleet with more than a handful of trucks, and at some point the workaround cost more than the fix.

If you're reading this, you already know the pain points — double-booked trucks, a RateCon that got overwritten, a driver settlement that doesn't match because someone fat-fingered a rate. What you probably don't have is a clean path off the thing. So here's one, week by week, including where it gets messy.

## Week 1: Find Out What Your Spreadsheet Actually Does

Before you touch any software, sit down with every dispatcher who touches the file and ask them to walk through it. Not what it's supposed to do — what they actually use it for. You'll find things nobody documented: a hidden column tracking detention hours, a color code for loads with a factoring company attached, a tab someone uses to track deadhead between drop and next pickup because the "real" numbers live somewhere else.

Write it all down. This is the map you'll need when you pick software, because the tool that replaces your spreadsheet has to cover what your team is actually doing, not what the org chart says they're doing.

## Week 2: Run the New System Alongside the Old One

Don't cut over cold. Pick your software and load a slice of real data — current week's loads, active drivers, open customers — and run it side by side with the spreadsheet for a week. This is where you find out what breaks first.

What usually breaks: someone keeps entering loads in the spreadsheet because it's muscle memory, and now you've got two sources of truth. The fix isn't more training, it's picking one dispatcher to own the cutover and telling the rest of the team the spreadsheet is read-only starting now. Also expect pushback from whoever built the original formulas — they've got equity in that thing, and they'll find reasons the new system "doesn't do it right." Sometimes they're right. Write those down too.

This is usually the point where a drag-and-drop dispatch board earns its keep. Dispatchers who've spent years dragging load numbers between spreadsheet cells adapt fast to dragging a load card from one truck to another — it's the same mental motion, minus the broken formulas and the version-control chaos of five people editing one file over VPN.

## Week 3: Move Fleet and Driver Data, Then Turn On GPS

This week is about trucks and drivers — units, trailers, driver contact info, CDL and medical card expirations, whatever compliance tracking you were doing manually or, more likely, not doing consistently enough. If you're on ELD hardware already, connect it now so you've got live location data instead of relying on check calls for basic "where's my truck" questions.

What breaks here: drivers who've never used a tracking link get nervous about it, and some will ask why. Have the answer ready — it's for ETA accuracy and detention documentation, not surveillance. Dispatchers get a real location feed, drivers still control their own schedule. Once it's running, you'll notice check calls drop off for anything routine; you only need to call when something's actually wrong.

## Week 4: Migrate Customers, Lanes, and Rate History

Pull every customer's rate history, accessorial terms, and lane preferences out of the spreadsheet tabs where they've been living and into the new system. This is tedious and there's no shortcut — someone has to sit with both screens open and move it line by line, or export/import if your format cooperates.

What breaks: rate inconsistencies you didn't know you had. Spreadsheets hide this well because every tab has its own version of "the rate." When it's all in one place, you'll find the customer who's been paying a rate from eighteen months ago because nobody updated the tab after the last negotiation. Annoying to find, good to find.

## Week 5: Cut Over Settlements and Invoicing

This is the week people get nervous, because it touches money. Run one full settlement cycle in parallel — old spreadsheet math against new system math — before you trust the new numbers alone. Check driver pay against mileage, deductions, and any advances. Check carrier settlements if you're brokering loads out. Check that customer invoices match the signed RateCon and that PODs are actually attached, not just referenced in a filename convention only one person understood.

What breaks: any custom pay structure that lived in a formula instead of a rule — percentage of load minus fuel surcharge, minus escrow, plus a stop-off fee that only applies past the third stop. These need to get rebuilt as explicit settings in the new system, not recreated as another spreadsheet bolted onto the TMS. That defeats the whole point.

## Week 6: Bring In EDI and Go Live for Real

If you're running tenders through EDI X12 204 with brokers or shippers, this is the week to turn that on and stop manually re-keying tender data into your board. Once it's flowing, retire the spreadsheet completely — don't leave it "just in case," because someone will use it in case, and you're back to two sources of truth.

## Why We Built It This Way

We built Techvia TMS because the carriers and brokers we talk to weren't looking for something complicated — they wanted the dispatch board, the settlements, the GPS tracking, and the EDI tenders in one place, without paying more every time they added a truck or hired a dispatcher. That's why it's a flat $199 a month for unlimited users, whether you're running 10 power units or 75. No per-seat math to do every time the fleet grows.

If your spreadsheet is starting to feel like the thing that runs your business instead of a tool that helps you run it, that's usually the sign it's time. You can try Techvia TMS free for 30 days, no credit card required, and run it side by side with what you've got now the same way we laid out above. Take a look at [Techvia TMS](https://www.techvia.software/products/tms) and see what week one looks like for your operation.