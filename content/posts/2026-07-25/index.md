---
title: "Seek & Destroy"
description: "Searching, replacing, and why Emacs does both better"
date: 2026-07-25T00:20:00+02:00
math: false
license: "CC BY-NC-SA 4.0"
hidden: false
comments: true
draft: false
tags:
- Software
- Emacs
categories:
- Emacs
series:
- Emacs, King of Editors
---

Every respectable text editor supports searching and replacing text, whether interactively or automatically. Editors from the upper shelves can also work with regular expressions.

And then there is the editor from the highest shelf — Emacs — which handles all of this brilliantly, in several different ways.

# Searching

In an average text editor, searching for text requires pressing a keyboard shortcut. Alternatively, you can try to remember beneath which pile of debris on your desk the mouse is buried, move it around until the pointer appears, and click whichever button opens the appropriate dialog box.

Then you type the search term, confirm it, and the text is found. Thank you very much, and we hope to see you again during your next search.

*But not in Emacs.*

By default, the King of Editors provides **incremental search**. Pressing `C-s` — or invoking `M-x isearch-forward` — starts the search and waits for input in the minibuffer.

With every character you type, Emacs attempts to find the entire string entered so far and moves the cursor to the matching location in the text. The current match is highlighted by default. Any other occurrences visible in the buffer are highlighted as well, using a different combination of foreground and background colours.

The search string is remembered between searches. Pressing `C-s` again moves the cursor to the next occurrence of the current search string.

When Emacs reaches the end of the buffer, it displays an appropriate message in the minibuffer. Pressing `C-s` once more wraps the search around and continues from the beginning of the buffer.

## Searching backwards

Incremental search also works **backwards**.

The `C-r` shortcut — or `M-x isearch-backward` — behaves just like forward incremental search, except that it moves in the opposite direction.

After reaching the *beginning* of the buffer, Emacs displays an appropriate message in the minibuffer. Pressing `C-r` again wraps the search around and continues from the *end of the buffer*.

## Search results

Pressing `M-s o` during an incremental search — which invokes `M-x isearch-occur` — opens another *window* containing the search results: every line in which the requested phrase occurs.

The matching phrase is, naturally, highlighted so that it stands out. At the top of the window, Emacs displays a summary showing how many matches were found, on how many lines, and in which buffer.

More importantly, the results are navigable. Move to any entry and press `RET`, and Emacs takes you directly to the corresponding line in the original buffer.

## Non-incremental search

What if someone simply wants to enter a complete search string first and then jump to its first occurrence?

A traditional, non-incremental search can be performed as follows:

* `C-s RET <search text> RET` — or `M-x search-forward` — searches forwards,
* `C-r RET <search text> RET` — or `M-x search-backward` — searches backwards.

In other words, it works much like incremental search, except that you press `RET` before entering the phrase.

## Case sensitivity

By default, Emacs uses a convenient mechanism known as **case folding**.

As long as the search string contains only lowercase letters, Emacs ignores case and attempts to match as many occurrences as possible. As soon as you enter an uppercase letter, the search becomes case-sensitive.

During an incremental search, `M-c` toggles case sensitivity explicitly, so there is no need to introduce an uppercase letter merely to change the matching behaviour.

## Searching with regular expressions

Emacs also provides both incremental and non-incremental searches based on regular expressions:

* `M-x re-search-forward` searches forwards for the next piece of text matching the given regular expression. When a match is found, the cursor is moved to its end. If no match exists, Emacs displays an appropriate message in the minibuffer.
* `M-x re-search-backward` performs the same operation in the opposite direction.
* `C-M-s` — `M-x isearch-forward-regexp` — starts an incremental forward search using a regular expression.
* `C-M-r` — `M-x isearch-backward-regexp` — starts an incremental backward search using a regular expression.

## Searching within a region

By default, Emacs searches through the buffer. To limit a search to a particular region, one can use a small but extremely useful trick: **narrowing**.

Press `C-x n n` — or invoke `M-x narrow-to-region` — to narrow the buffer to the currently selected region.

This is interesting because the rest of the text temporarily disappears. All subsequent operations, including searching, are restricted to the visible part of the buffer.

Once the work is complete, press `C-x n w` — or invoke `M-x widen` — to restore the hidden text to its rightful place.

## Search history

Search history is, of course, preserved.

While searching, you can use:

* `M-p` to recall the previous search string,
* `M-n` to move to the next search string in the history.

## Other useful commands

* `C-y` — `M-x yank` — inserts the most recently killed text into the search string, because why would it not?
* `C-w` adds the remainder of the word at point to the search string.

# Replacing

Once again, the King of Editors refuses to blend in with the crowd and offers something better than the competition.

`M-%` — `M-x query-replace` — interactively replaces occurrences of one string with another, starting from the position at which the command was invoked.

At every match, Emacs asks the user what should happen next:

* `y` replaces the current match and moves to the next one,
* `n` skips the current match and moves to the next one,
* `q` ends the replacement operation,
* `!` replaces the current match and every remaining match without asking any more unnecessary questions.

Other available actions can be found, for example, by invoking:

```text
M-x describe-function RET query-replace RET
```

`C-M-%` — `M-x query-replace-regexp` — works in the same way, but uses regular expressions.

Emacs also tries to preserve the **capitalisation style** of the replaced text.

Suppose that every occurrence of `windows` must be replaced with `linux`. Emacs may perform the following replacements:

* `windows` becomes `linux`,
* `Windows` becomes `Linux`,
* `WINDOWS` becomes `LINUX`.

> The author believes that Windows should be replaced with Linux not only in this example, but also in the lives of us all.

## Non-interactive replacement

Emacs also provides commands that replace all matching occurrences without asking for confirmation:

* `M-x replace-string` replaces every occurrence of one string with another, starting from the position at which the command was invoked.
* `M-x replace-regexp` replaces every piece of text matching the given regular expression with the specified replacement string, again starting from the current position.

## Replacing within a region

If a region is active, Emacs performs replacement commands within that region by default.

There is no need to narrow or widen the buffer manually — although, naturally, Emacs will not prevent anyone from doing so.
