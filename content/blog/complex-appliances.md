+++
title = "Goodbye Non-KISS Appliances"
description = "A rant about complex appliances."
date = 2023-12-14
draft = false

[taxonomies]
categories = ["Lifestyle"]
tags = ["rant", "kiss"]

[extra]
toc = true
copy = false
math = false
mermaid = false
+++

Yesterday, we gave away our ~6-year-old GE French-door refrigerator after a
never-ending slog of repairs. At least the guy and his mother who came to pick
it up have a side-hustle of repairing and reselling appliances. There's hope it
won't end up in a landfill. He offered to sell me a 4-door Samsung. No Thanks!
I'm done with over-complicated appliances. Refrigerator repair is not a skill
I'm looking to hone any further.

<!-- more -->

# Survivorship bias

I have a refrigerator in my garage that's a side-by-side style Kenmore (Model#
25354362402). The sticker on the inside indicates a manufacture date of `06-04`.
It was bought as a scratch-and-dent model from Sears Outlet. At some point, over
a decade ago, the relay on the compressor went out. The fix was cheap and easy.
Later, we started having problems with the ice maker / dispenser. Instead of
fixing it, we decided to get a new one and use the side-by-side as a second
refrigerator in the garage. Since then, it's out-lived two French-door
refrigerators we have had in the intervening time.

# The French-door woes begin

For the replacement, we went with a French-door style Kenmore (Model#
79572053110) because, you know, all that space in the fridge, and again scratch
/ dent sale.

I don't even remember all the problems we had with it, and this was before I
started keeping better records. I have the notes from the last repair receipt:

>cked unit ice build up around ice fan motor fines removed heater good no problem found.

After about the second repair of the same issue, I was surprised they let me buy
an extended warranty on it. Fast-forward past 6-10 (I forget) repairs /
replacements of the same part and much hassle, they gave me store credit as
though I bought it new without dents. The amount, I forget, but ended up being a
curse because we felt compelled to use it all on an even fancier French-door
refrigerator this time brand new.

I wish I could go back in time and slap myself in the face for deciding to buy
another French-door refrigerator. I even remember thinking about the [KISS
principle](https://en.wikipedia.org/wiki/KISS_principle) at the time, but
somehow thought switching brands and spending 3x the amount would save us from
the complexity.

# Enter the GE Café

This refrigerator (model# CFE28TSHESS) sports all these *wonderful* features:

- touch screen LCD
- ice, water, and hot water dispenser
- RFID chipped water filters to lock-in customers to buy their $50 filters

I keep notes and `TODO` tasks using [org-mode](https://orgmode.org/), and I was
just gathering my notes on this refrigerator and found I have written ~1300
words on it, not counting this blog post. The following is a summary.

The first annoyance was having to employ [a
hack](https://www.youtube.com/watch?v=ellbAY9IvgU) to get around GE water filter
lock-in.

I've had the evaporator fan fail on me twice. Once in 2020 and its replacement
failed in 2022. When that happens, the fridge compartment doesn't cool well. A
[video on how to replace it](https://www.youtube.com/watch?v=IeG4CVkAFgs&t=58s)
for context.

Ice builds up in the bottom of the freezer if the drain to the evaporation pan
gets clogged. Cleaning the rubber hose and dust off the condenser is a yearly
chore at least. It becomes the reason to take all the food out and fully clean
everything. Otherwise, get ready to spend hours defrosting. One hack we found to
speed it up was to pour nearly boiling water down the drain to defrost the ice
build up.

Once the freezer stopped cooling, and we noticed the case was hot to the touch
on the side between the freezer and fridge compartments. This ended up being due
to a failed condenser fan. A real PITA to replace, by-the-way.

Around that time I found the technical service manual online and realized one
can view error codes and run all sorts of tests via the LCD touch screen. Then I
found out that the touch screen wasn't working. Since a new touch screen LCD
assembly is over $300, so I ordered one off ebay instead.

We had a problem with the "new" LCD assembly intermittently pressing the hot
water button on its own. Basically, the person I got the part from on ebay lied
saying that it was brand new. It was refurbished. We could tell because there
had been some soldering work done to the board. Instead of uninstalling it and
returning it, we realized that button pressing over and over wasn't affecting
any functionality (we don't use its hot water dispenser anyway). We just turned
the sound off to stop the annoying repeated chiming noise.

From running fan tests in service mode, I could tell the odor remover fan also
died, but I never cared to replace it. Fans are expensive!

Recently, it stopped working again. With the power of YouTube and a multi-meter
I was able to test the capacitor on the relay was good, but the relay was bad.
Replaced the relay, but the compressor still is not starting. Tested that it is
getting 120VAC.

Here's my notes from my before replacing the relay:

```
- Note taken on [2023-12-05 Tue 20:09] \\
  WR07X10131 it was the relay that plugs into the compressor. It rattles when
  shaken, and it doesn't have continuity between the two main pins.

  | Fault Code Definition                              | code | count | days ago |
  |----------------------------------------------------+------+-------+----------|
  | Cold Water Cup Switch input missing                |  709 |    14 |        2 |
  | Condenser Fan cannot reach target RPM              |  105 |     2 |        0 |
  | FF Temp Exceeded 75°F                              |  303 |     1 |        0 |
  | FZ Temp Exceeded 72.5°                             |  304 |     1 |        1 |
  | Condenser Fan cannot reach target RPM              |  105 |     2 |        0 |
  | N/A ????                                           |  800 |    63 |        0 |
  | 5 consecutive FF abnormal defrosts.                |  203 |   244 |        1 |
  | N/A ????                                           |  803 |   204 |        0 |
  | N/A ????                                           |  600 |     2 |       13 |
  | Deli Pan Fan Feedback Missing when fan is running. |  112 |   120 |        0 |
  | FF Defrost Heater on for max time.                 |  201 |   217 |        0 |
```

Note the `Condenser Fan cannot reach target RPM` despite it being recently
cleaned out with an air compressor and replaced on `2021-06-05`, the $74 fan can't
reach RPM? It was working and has nothing to do with the compressor not running,
but it's hard to justify spending more money to fix this thing with the non-stop
fan issues.

At the limit of my refrigerator repair skills and wits' end, we decided to give
away the GE refrigerator.

# Keep it simple, stupid!

Based on [this video](https://www.youtube.com/watch?v=rKJgYVhZ6-w), We are
upgrading to simplicity. Top freezer refrigerator. ~~I'll get an ice maker
add-on~~ (no dispenser) and a water pitcher for filtered water. After using ice
trays for a few weeks, we decided it's actually not that bad and returned the
ice maker.

We haven't decided on brand / model yet. I understand that the long-lasting
refrigerator in my garage is probably a coincidence.

I'll admit engineering effort can tame complexity. For example, the combustion
engine and hybrid cars are surprisingly reliable. And perhaps, for the right
price, there are brands that have pulled it off with complex refrigerators.
There's just not enough value added by any of the fancy features to warrant the
potential headache.

# Other appliance anecdotes

We're at the end of a long year of remodeling. There are lot of stories I could
tell, but I feel like it's worth a mention to how painless it is to live without
a garbage disposal. Another thing in my life I don't have to worry about.

Old-school top load washing machine has been serving us well for the most part
(Maytag MVWP575GW). The shift actuator assembly did go bad once. It was about
$60, but wasn't too hard to replace.

We have a Whirlpool dryer since about 2004 too. I replaced the heating element
once a few years ago. Obviously, no extraneous features to worry about there.

# Discussion

- <https://news.ycombinator.com/item?id=39238010>
- <https://slrpnk.net/post/4915674>
