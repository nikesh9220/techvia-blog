---
title: "freight invoicing and settlement basics"
date: 2026-09-14
draft: false
tags: ["invoicing","settlements","cash flow","freight billing"]
categories: ["TMS"]
description: "A load-by-load walkthrough of how freight money moves from delivery to cash in the bank, and every point along the way it quietly leaks."
showToc: true
---

Every carrier and broker we talk to can tell you their revenue per load. Fewer can tell you, without pulling three systems and a spreadsheet, how many days actually pass between delivery and cash landing in the bank. That gap is where the money leaks. Not in some dramatic fraud sense — in a hundred small delays, each one shaving margin off a load that already ran thin.

This is the plain version of freight invoicing and settlement: what happens after the truck backs out of the dock, who touches the paperwork, and where the process breaks on a normal week, not just a bad one.

## The load isn't done when the truck is empty

A driver dropping the last pallet feels like the finish line. It isn't. For a broker, the load isn't closed until the invoice is built, sent, and paid. For a carrier, it isn't closed until the driver or owner-operator has been settled correctly against that same load. Two different clocks start ticking the moment the trailer doors close, and both of them cost you money if they run long.

The sequence, in order, looks like this:

- Delivery happens, driver gets a signed POD
- POD and any lumper or detention paperwork gets back to the office
- Invoice is built against the original rate confirmation and sent to the customer or factor
- Customer pays on terms, or the invoice ages past terms
- Driver or carrier settlement is calculated and paid out

Every one of those five steps has a failure mode. Let's walk them.

## The POD is the whole invoice, not a formality

You cannot invoice cleanly without a proof of delivery, and you cannot dispute a detention claim without one either. A driver who forwards a blurry photo three days late doesn't just delay billing — it delays billing for every load stacked behind it in the queue, because most back offices process PODs in the order they arrive, not the order the loads ran.

The dispatchers running this well don't leave POD collection to memory. They build it into the check call rhythm: confirm delivery, confirm the POD is in hand or on its way, before the truck gets its next dispatch. The ones running it badly find out on day 28 that a load from three weeks ago never got billed because nobody flagged the missing paperwork.

## Rate confirmation mismatches are the quiet killer

The RateCon is the contract for that specific move — origin, destination, stops, rate, accessorials agreed to up front. When the invoice doesn't match the RateCon line for line, you get a short-pay or a dispute, and disputes take weeks, not days, to resolve.

The usual mismatch isn't the linehaul rate. It's the extras. A detention charge that was verbally approved but never written into the RateCon. A second pickup that got added after the load was booked but never got documented as a stop change. An extra stop fee nobody remembered to add to the invoice. Every one of these is legitimate money the carrier earned and the broker owes — and every one of them gets contested or ignored if it isn't tied back to paperwork the customer already agreed to.

## Detention and accessorials: bill them or eat them

Detention is the accessorial everyone talks about and almost nobody bills consistently. If a driver sits two hours past free time at a shipper, that's billable — but only if someone logged the arrival time, the departure time, and got it into the invoice before it went out. Once an invoice is sent without the detention line, most customers won't pay it retroactively. You get one shot.

The same goes for lumper fees, layover pay, and extra stops. These are small dollar amounts per load, but they compound across a fleet running dozens of loads a week. A carrier that consistently under-bills accessorials isn't losing money on one bad load — they're losing a percentage point of margin on every load, quietly, forever.

## Settlements run on a different clock than invoicing

Here's where carriers specifically get squeezed: driver and owner-operator settlements often need to go out on a fixed weekly or biweekly schedule regardless of whether the customer has paid yet. That means the carrier is fronting cash on loads that haven't been collected on. Every day an invoice sits unpaid past terms is a day the carrier is carrying that float out of pocket.

This is why settlement accuracy matters as much as invoice accuracy. A driver settlement built off the wrong mileage, the wrong percentage, or a missed deduction doesn't just create a payroll dispute — it creates a driver who starts wondering if the numbers can be trusted at all, which is a retention problem dressed up as a math problem.

## Where the process actually breaks

Most of the leakage we've described traces back to one root cause: the POD, the RateCon, the invoice, and the settlement all live in different places, updated by different people, on different timelines. Someone has to manually carry information from one to the next, and manual carrying is where numbers get dropped, forgotten, or entered wrong.

We built Techvia TMS's invoicing module around this exact problem — tying the invoice directly back to the load, the RateCon terms, and the POD that closed it, so the accessorials that were earned actually make it onto the bill, and the settlement calculates off the same numbers instead of a second, separately-typed version of them. It doesn't replace the check call or the driver who needs to send that POD photo on time. It just means the paperwork that already exists gets used correctly instead of re-keyed three times before it turns into cash.

## Start with one week

You don't need new software to start fixing this. Pull last week's loads and time each one from delivery to invoice sent, and from invoice sent to payment received. Look at how many loads had a detention or extra-stop charge that never made it onto the bill. That single exercise will tell you more about where your money is leaking than any benchmark someone hands you.

If you want the load-to-invoice-to-settlement chain running off one shared set of numbers instead of three, take a look at [Techvia TMS](https://www.techvia.software/products/tms). The 30-day trial doesn't ask for a credit card, so you can run it against a real week of your own loads before deciding anything.