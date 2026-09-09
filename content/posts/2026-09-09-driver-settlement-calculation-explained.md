---
title: "Driver Settlement Calculation Explained"
date: 2026-09-09
draft: false
tags: ["driver settlements","trucking payroll","carrier operations","dispatch"]
categories: ["Trucking"]
description: "A plain-language walkthrough of per-mile, percentage, and escrow settlement math, with worked examples and the deductions that trip carriers up."
showToc: true
---

Ask five dispatchers how they calculate a driver's settlement and you'll get five slightly different answers, and at least one of them will be wrong in a way that costs the carrier money or the driver's trust. We built Techvia TMS because the settlement step is where a lot of small fleets quietly bleed accuracy — not because anyone's being careless, but because the math has more moving parts than it looks like from the outside.

This post walks through the three common pay structures — per-mile, percentage, and escrow-backed — with real numbers, and then covers the deductions that most often get miscalculated or forgotten.

## Why the settlement is the whole relationship

A rate confirmation tells a driver what the load pays the carrier. A settlement tells the driver what they actually take home. Those are two different documents built from two different sets of numbers, and drivers know the difference immediately when a settlement doesn't match what they expected to see. Get it wrong twice and you're recruiting again next month.

The fix isn't more spreadsheets. It's a consistent method, applied the same way every pay period, with every input traceable back to a load, a rate con, and a POD.

## Per-mile pay: simple math, complicated inputs

Per-mile settlements look like the easiest math in trucking. Miles times rate equals pay. The trouble is almost never the multiplication — it's what counts as a mile.

Say a driver runs a load from Dallas to Charlotte, 936 loaded miles, at $0.62 per mile.

- Loaded miles: 936 × $0.62 = $580.32

Now add the deadhead. The driver had to run 84 empty miles from their previous drop to the Dallas pickup. If your pay policy covers deadhead at the same rate, that's another $52.08. If deadhead is paid at a reduced rate — say $0.20/mile — it's $16.80 instead. Either way, the number has to be written into the driver's pay policy and applied the same way every time, because "sometimes we pay deadhead and sometimes we don't" is how you end up with a driver comparing settlements with a coworker and calling you out on it.

Stop-off pay is the other place per-mile settlements go sideways. If that Dallas-to-Charlotte run had a second pickup in Memphis, and your policy pays $50 per additional stop, that's another line item that has to make it onto the settlement — not get absorbed into "miscellaneous" or forgotten because the dispatcher who booked the load isn't the one who runs payroll.

## Percentage pay: the split that follows the rate

Percentage-of-revenue settlements are common with owner-operators and lease drivers, and the math changes the moment the linehaul rate changes — which is exactly why these settlements need to be tied directly to the rate con, not to a dispatcher's memory of what the load paid.

Take a load that billed at $2,400 linehaul, with the driver on a 72% split.

- $2,400 × 0.72 = $1,728 gross driver pay

Now add a $150 detention charge the carrier collected from the broker for four hours of wait time at the receiver, over the standard two free hours. If your percentage applies to total revenue including detention, the driver gets 72% of $2,550, or $1,836. If detention is paid flat, separate from the percentage split, the driver gets $1,728 plus $150, or $1,878. Those two methods produce different numbers on the same load, and the only way to avoid an argument is to write the policy down and apply it the same way on every settlement, every time.

This is also where fuel surcharge trips people up. If the $2,400 linehaul figure already includes FSC, and the driver's percentage applies to the whole number, that's one calculation. If FSC is paid separately and outside the percentage split — common when a carrier wants drivers to see fuel money as reimbursement, not commission — that's a different math path entirely. Neither approach is wrong. Inconsistency is what's wrong.

## Escrow: the safety net that needs its own math

Escrow accounts protect the carrier against cargo claims, accidents, and equipment damage, and most lease-purchase and owner-operator agreements specify a target balance — commonly somewhere in the $1,000 to $2,500 range depending on the agreement, though the number itself is whatever your contract says, not a market standard.

Say the target escrow balance is $2,000 and the driver's current balance is $1,400. The agreement calls for $75 withheld per settlement until the target is met.

- Gross settlement before escrow: $1,878
- Escrow withheld: $75
- Net after escrow: $1,803

Once the balance hits $2,000, the withholding stops — and this is the step carriers most often forget to automate. A driver who keeps getting $75 pulled after they've already hit target isn't going to assume it's an accounting glitch. They're going to assume you're skimming, and that conversation is a hard one to walk back.

## Deductions: where trust gets tested

Beyond escrow, a typical settlement carries several other deductions, and each one needs a paper trail the driver can see:

- **Fuel advances or fuel card usage** — tied to actual card transactions, not an estimate
- **Cargo insurance or occupational accident premiums** — a fixed weekly or per-load amount specified in the driver's agreement
- **Equipment lease payments** — for lease-purchase drivers, due on a fixed schedule regardless of miles run
- **Advances against future settlements** — should show the original advance and the repayment on the same settlement, not just a mystery negative number
- **Chargebacks for claims or damage** — should reference the specific incident and load number, never a lump "misc" deduction

Every one of those deductions should trace back to a document — a fuel receipt, a signed lease, a claim file. A driver who can't see why $340 disappeared from their check is a driver who starts shopping other carriers, and turnover costs a lot more than the disputed $340 ever did.

## Putting it together

A clean settlement shows gross pay by load, itemized additions like detention and stop pay, itemized deductions with references, the escrow line if applicable, and a net figure that a driver can check against their own log of miles and stops without a calculator. If a driver has to call you to understand their own paycheck, the settlement isn't doing its job.

This is exactly the kind of calculation that gets error-prone fast once you're running 20, 40, or 75 trucks with a mix of per-mile and percentage drivers, different deadhead policies, and escrow accounts at different stages. Techvia TMS handles driver and carrier settlements alongside dispatch, invoicing, and compliance, so the math runs consistently off the same load and rate con data every time — no separate spreadsheet reconciling against a dispatch board that's already moved on to next week's loads.

If your settlements are still built by hand at the end of each pay period, it's worth seeing what a system built specifically for this looks like. You can check out [Techvia TMS](https://www.techvia.software/products/tms) and run it free for 30 days, no credit card required, on your own drivers and your own numbers.