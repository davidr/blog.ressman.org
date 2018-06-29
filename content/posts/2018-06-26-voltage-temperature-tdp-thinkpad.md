---
date: 2018-06-26
lastmod: 2018-06-26
title: "Notes on thermal throttling, undervolting, and TDP regulation"
description: "or... how to break a laptop with Go without really trying."
authors: ["davidr"]
categories:
  - notes
tags:
  - thinkpad
  - msr
  - undervolt
  - x1c6
  - golang
toc: true
---

# Background

It turns out that I know next to nothing about how modern consumer hardware works at a low level. 
I bought a schnazzy new ThinkPad X1C6 and while trying to iron out some brand-new-laptop Linux
issues, I spent some time looking at its thermal management. It runs a little hotter than my last
one, so I thought I'd have some fun and work it out.

It also turns out that although the temperature throttling and TDP tuning are well-documented, but
in spite of Intel releasing their
[XTU](https://downloadcenter.intel.com/download/24075/Intel-Extreme-Tuning-Utility-Intel-XTU-) 
tool, they don't actually seem to document the process for altering the cpu voltage *at all*. There
are a few smart people out there who've reverse engineered the MSR activity from XTU, so that's pretty
much all there is to work from.

It's been a while since I did any bit twiddling and I'm trying to spin up on kubernetes development,
so it seemed like as good any to dust off my Go chops and play around.

# Model-Specific Registers (MSRs)

To get anything done with the CPUs, we need to use the MSR devices (`/dev/cpu/$CPUNUM/msr`)

For the MSRs that are documented, 
[this seems to be the best place](// https://software.intel.com/sites/default/files/managed/22/0d/335592-sdm-vol-4.pdf).

## Throttle Temperature Control (msr 0x1a2)

Bits | Field
-----|------
63:28|Reserved
27:24|TCC Activation Offset (read/write)
23:16|Temperature Target (read only)
15:0 |Reserved

## PKG RAPL Power Limit Control (msr 0x610)

This one is split into two parts. `0x610` for the power limit control, and `0x606` for the unit
definition for the settings in `0x610`.

### 0x610

Bits | Field
-----|------
63:24|Reserved
23:17|Time window (s)
16   |Package Clamping Limitation
15   |Enable Power Limit
14:0 |Package Power Limit

### 0x606

Bits | Field
-----|------
63:20|Reserved
19:16|Time Unit (Time related information (in seconds) is in unit of 1S/2^TU; where TU is an unsigned integer represented by bits 19:16. Default value is 1010b, indicating power unit is in 0.977 millisecond.)
15:13|Reserved
12:8 |Energy Status Units (Energy related information (in Joules) is in unit of 1Joule/ (2^ESU); where ESU is an unsigned integer represented by bits 12:8. Default value is 01110b, indicating energy unit is in 61 microJoules)
7:4  |Reserved
3:0  |Power Units (Power related information (in Watts) is in unit of 1W/2^PU; where PU is an unsigned integer represented by bits 3:0. Default value is 1000b, indicating power unit is in 3.9 milliWatts increment)

## Voltage Offset (msr 0x150)

Since this isn't documented, this is where it gets a little tricky. From the available information out there
by people smarter than me who've looked into this, here's what I've been able to put together. This is probably
8 different kinds of wrong.

#### Example value:
```
63    56 55    48 47    40 39    32 31    24 23    16 15     8 7      0
10000000 00000000 00000010 00010001 11101100 11000000 00000000 00000000
   8   0    0   0    0   2    1   1    e   c    c   0    0   0    0   0
```

Bits |Field
-----|-----
63   |Unknown, but seems like it must be set to 1
43:40|Voltage Plane
36   |Unknown, but seems like it must be set to 1
32   |Write bit (set to 1 if writing value)
31:21|Voltage offset value (signed)

# Next steps:

I'm writing [some code](https://github.com/davidr/ddtp) to use the above.

# See also:

* https://blog.mthode.org/posts/2018/Jan/undervolting-your-cpu-for-fun-and-profit/
* https://github.com/mihic/linux-intel-undervolt
