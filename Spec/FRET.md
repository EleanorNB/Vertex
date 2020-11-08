Find Resources, Effect Tasks
----------------------------
A consistent and eager system shell.

## Overview
Unlike most shells, FRET is designed to perform the full function of a command line -- an application should not even be able to tell how it was invoked, and changing its context should be as simple as passing different arguments, no `chroot` required. To this end, it relies more on sigils than is typical, and cooperates tightly with AMP.

## Startup
FRET is typically invoked by STAR at system start, but this is not required. It requires root scheme and allocator capabilities, as well as channels for input (stream, in), output (stream, out), log (stream, out), and interface (terminal).

## Channels
