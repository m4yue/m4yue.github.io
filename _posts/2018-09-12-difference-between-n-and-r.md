---
layout: default
title: Difference between \n and \r?
---

## Difference between \n and \r?

In terms of ascii code, it's 3 -- since they're 10 and 13 respectively;-).

But seriously, there are many:

  - in Unix and all Unix-like systems, \n is the code for end-of-line, \r means nothing special as a consequence, in C and most languages that somehow copy it (even remotely), \n is the standard escape sequence for end of line (translated to/from OS-specific sequences as needed)
  - in old Mac systems (pre-OS X), \r was the code for end-of-line instead
  - in Windows (and many old OSs), the code for end of line is 2 characters, \r\n, in this order
  - as a (surprising;-) consequence (harking back to OSs much older than Windows), \r\n is the standard line-termination for text formats on the Internet
  - for electromechanical teletype-like "terminals", \r commands the carriage to go back leftwards until it hits the leftmost stop (a slow operation), \n commands the roller to roll up one line (a much faster operation) -- that's the reason you always have \r before \n, so that the roller can move while the carriage is still going leftwards!-) [Wikipedia](https://en.wikipedia.org/wiki/Newline#History) has a more detailed explanation.
  - for character-mode terminals (typically emulating even-older printing ones as above), in raw mode, \r and \n act similarly (except both in terms of the cursor, as there is no carriage or roller;-)

In practice, in the modern context of writing to a text file, you should always use \n (the underlying runtime will translate that if you're on a weird OS, e.g., Windows;-).

The only reason to use \r is if you're writing to a character terminal (or more likely a "console window" emulating it) and want the next line you write to overwrite the last one you just wrote (sometimes used for goofy "ascii animation" effects of e.g. progress bars) -- this is getting pretty obsolete in a world of GUIs, though;-).
