---
title: "Lisp — What's It All About?"
description: "The absolute basics of Emacs Lisp needed to configure the Editor"
date: 2026-10-08T14:50:00+02:00
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

For more advanced operations in the One True Editor, it is worth learning at least a little Lisp — specifically **Emacs Lisp**, usually shortened to **Elisp**.

It is the primary language used for Emacs extensions, packages, and configuration.

A remarkable number of recipes found on the Internet boil down to:

> Paste the following lines into this particular file somewhere in your home directory and reload Emacs.

And that is fine.

You can survive for years that way.

Still, it may be useful to go a little deeper and understand at least **what we are actually pasting, where, and why**.

Besides, if we are already dismantling the King of Editors piece by piece, completely ignoring Lisp would be rather difficult.

Anyone firmly committed to the "just tell me what to paste and where" school of thought may, of course, skip the following section.

Or rather: wall of text.

# A World That Will Never Return

Long ago, programming languages had simple, readable syntax, and you could tell what a program did merely by looking at it from across the room.

For example:

```asm
.section .data
msg:
    .ascii "Hello, Cruel World!\n"
    msg_len = . - msg

.section .text
.globl _start

_start:
    movl $4, %eax
    movl $1, %ebx
    movl $msg, %ecx
    movl $msg_len, %edx
    int $0x80

    movl $1, %eax
    xorl %ebx, %ebx
    int $0x80
```

Simple.

Readable.

Self-explanatory.

Right?

Things could have stayed that way, but a rather large number of high-level languages appear to disagree.

The history of Lisp inside Emacs is quite interesting.

The original Emacs was not written in Lisp. It began life as a collection of macros for **TECO**.

Later, **Bernie Greenberg** wrote a version of Emacs for Multics in **MacLisp**. It demonstrated that using an actual programming language as the extension language for an editor was considerably more pleasant than endlessly expanding a macro system.

Richard Stallman later recalled an interesting side effect.

Secretaries in Greenberg's office started writing extensions for Emacs themselves.

They had been given a manual explaining how to extend the editor, but the manual never told them that what they were doing was _programming_.

So, unaware that they supposedly could not program, they read the documentation...

...and learned to program.

Beautiful.

Lisp then remained with Emacs for the following decades.

We may as well get acquainted.

# Language Basics

Let us begin at the beginning.

And since every programming language must begin by greeting this cruel world, there is no reason to break with tradition:

```lisp
(defun hello-world ()
  "A function that greets the world."
  (interactive)
  (message "Hello, cruel world"))
```

The first thing you notice?

Parentheses.

Lots of parentheses.

We will get used to them.

What do we have here?

- `defun` — a function definition,
- `hello-world` — its name,
- `()` — the argument list, empty in this case,
- `"A function that greets the world."` — the function's documentation,
- `(interactive)` — a declaration that the function may be invoked as an Emacs command,
- `(message ...)` — the actual body of the function.

## Running It

Paste the code into the **scratch** buffer.

It is an excellent place for first experiments with Elisp.

Put point **after the final parenthesis of the definition** and press:

```text
C-x C-e
```

which invokes:

```text
M-x eval-last-sexp
```

Emacs evaluates the preceding Lisp expression — an _S-expression_, usually shortened to **sexp** — and the function becomes available for the current session.

Now execute:

```text
M-x hello-world
```

and the echo area displays:

```text
Hello, cruel world
```

> Point matters when using `C-x C-e`. The command evaluates the expression immediately before it. Put point somewhere in the middle of the definition and you may evaluate a completely different fragment — or get an error.

What about that string immediately following the argument list?

It is the function's **docstring**.

We can now run:

```text
M-x describe-function RET hello-world RET
```

and Emacs will display documentation for our brand-new function.

# Arguments and `interactive`

How about an argument?

```lisp
(defun hello-someone (name)
  "Ask for a name and display a greeting."
  (interactive "sEnter your name: ")
  (message "Hello, %s. Lisp isn't that bad!" name))
```

This function accepts one argument: `name`.

`interactive` tells Emacs how to obtain that argument when the function is invoked interactively.

The letter:

```text
s
```

means a string.

So after:

```text
M-x hello-someone
```

Emacs asks:

```text
Enter your name:
```

and the entered text becomes the value of `name`.

Two arguments?

Certainly:

```lisp
(defun hello-someone (name age)
  "Ask for a name and age, then display a greeting."
  (interactive "sEnter your name: \nnAnd your age: ")
  (message "Hello, %s (%d). Lisp isn't that bad!" name age))
```

Here:

```text
s
```

means string, while:

```text
n
```

means number.

If we enter something that is not a number, Emacs keeps asking.

We may, naturally, surrender:

```text
C-g
```

A universal solution.

# Some Arithmetic

Let us try something slightly more useful:

```lisp
(defun multiply (a b)
  "Ask for two numbers and insert their product into the current buffer."
  (interactive "nEnter the first number: \nnEnter the second number: ")
  (let ((result (* a b)))
    (insert (format "%d" result))))
```

Here we encounter one of Lisp's most distinctive features:

```lisp
(* a b)
```

The operator appears **before its operands**.

This is known as **prefix notation**, or — appropriately enough — **Polish notation**.

Not Reverse Polish Notation.

That would look more like:

```text
a b *
```

The Polish edition of this article may therefore indulge in a small amount of national pride.

Without reversing anything.

The code also does two other interesting things.

First, it creates a local binding:

```lisp
(let ((result (* a b)))
  ...)
```

Then, instead of `message`, it uses:

```lisp
insert
```

which inserts text directly into the current buffer at point.

# Data Types

## Simple Variables

Variables may have global values:

```lisp
(setq my-variable 42)
(setq my-string "This is my string")
(setq my-boolean t)
```

`setq` assigns a value to a symbol.

We can also establish a local binding with `let`:

```lisp
(let ((my-variable 42))
  (message "Value: %d" my-variable))
```

The value `42` applies only inside the body of this `let`.

Boolean values are:

```text
t
```

for true, and:

```text
nil
```

for false.

`nil` is also the empty list.

This is not a coincidence.

This is Lisp.

## Lists

Lists are one of Lisp's fundamental data structures.

We can create one like this:

```lisp
(setq my-list '(1 2 3 4 5))
```

The apostrophe means:

> Do not evaluate this. Treat it as data.

Without it, Lisp would try to interpret:

```lisp
(1 2 3 4 5)
```

as a call to something named `1` with the arguments `2`, `3`, `4`, and `5`.

We may also construct a list explicitly:

```lisp
(setq my-list (list 1 2 3 4 5))
```

The classic list functions include:

- `car` — the first element,
- `cdr` — everything except the first element,
- `nth` — the element at a particular index.

For example:

```lisp
(car '(a b c))       ; => a
(cdr '(a b c))       ; => (b c)
(nth 2 '(a b c d))   ; => c
```

Indices, naturally, begin at zero.

As civilization intended.

## Modifying Lists

`setf` can assign a value to a location described by another expression:

```lisp
(setq my-list '(1 2 3 4 5))

(setf (nth 1 my-list) 70)
(setf (nth 2 my-list) "who knows")
```

Now:

```lisp
my-list
```

contains:

```lisp
(1 70 "who knows" 4 5)
```

An element can easily be added to the front:

```lisp
(push 0 my-list)
```

And `append` joins sequences:

```lisp
(append my-list '(6 7))
```

There is an important difference: `append` **returns a new list**. It does not automatically store it back in `my-list`.

To change the variable:

```lisp
(setq my-list (append my-list '(6 7)))
```

Naturally:

```lisp
(append '(a b) '(c d))
```

returns:

```lisp
(a b c d)
```

# Association Lists

An **alist**, or _association list_, is a list of key-value pairs.

For example:

```lisp
(setq my-dictionary
      '((number . 42)
        (identifier . "some string")))
```

It can be thought of as a very simple dictionary.

For example:

```lisp
(alist-get 'number my-dictionary)
```

returns:

```text
42
```

Alists appear in Emacs configuration remarkably often.

Much more often than you may initially wish.

Eventually it stops bothering you.

# Vectors

A vector is a fixed-length sequence:

```lisp
(setq my-vector [10 20 30])
```

An element can be retrieved with:

```lisp
(aref my-vector 1)
```

which returns:

```text
20
```

Unlike lists, vectors provide direct indexed access to their elements.

# Conditionals

## `if`

The basic conditional looks like this:

```lisp
(if condition
    when-true
  when-false)
```

For example:

```lisp
(if (> variable 42)
    (message "The value is too large!")
  (message "Still within limits."))
```

Once again, we see prefix notation:

```lisp
(> variable 42)
```

If the `then` branch needs to perform several forms, they can be grouped with:

```lisp
(progn
  (do-one-thing)
  (do-another)
  (do-a-third))
```

For example:

```lisp
(if (> variable 42)
    (progn
      (message "Too much!")
      (setq variable 42))
  (message "Fine."))
```

## `when` and `unless`

When no "otherwise" branch is needed, the code can often be made clearer.

`when`:

```lisp
(when (some-condition)
  (message "Condition satisfied!"))
```

and `unless`:

```lisp
(unless (some-condition)
  (message "Condition not satisfied. I can fix that!"))
```

`when` roughly means:

> execute the body if the condition is true,

while `unless` means:

> execute the body if the condition is false.

## `cond`

`cond` evaluates a sequence of conditions.

It resembles a more flexible `if` / `else if` chain and can sometimes fill a role similar to `switch` in other languages.

Each branch may contain an arbitrary condition:

```lisp
(defun check-number (x)
  (interactive "nEnter a number: ")
  (cond
   ((< x 0)
    (message "The number is negative."))
   ((= x 0)
    (message "The number is exactly zero."))
   ((> x 10)
    (message "The number is greater than 10."))
   (t
    (message "The number is positive, but no greater than 10."))))
```

The final:

```lisp
(t ...)
```

acts like a final `else`.

## Combining Conditions

Conditions can be combined with:

```text
and
or
not
```

For example:

```lisp
(and (> x 0) (< x 10))
```

or:

```lisp
(not (= x 42))
```

# Loops

## `while`

The basic loop is:

```lisp
(while condition
  ...)
```

For example:

```lisp
(let ((counter 3))
  (while (> counter 0)
    (insert (format "Countdown: %d...\n" counter))
    (setq counter (- counter 1))))
```

## `dolist`

`dolist` iterates over the elements of a list:

```lisp
(let ((text-editors
       '("Emacs" "And" "Then" "Nothing" "For" "A While")))
  (dolist (editor text-editors)
    (insert (format "* %s\n" editor))))
```

The ranking is, naturally, entirely objective.

## `dotimes`

`dotimes` executes its body a specified number of times and provides a counter beginning at zero:

```lisp
(dotimes (i 3)
  (insert (format "This is repetition number: %d\n" i)))
```

The result:

```text
This is repetition number: 0
This is repetition number: 1
This is repetition number: 2
```

Technically, `dotimes` is a macro.

Let us not worry yet about why it is a macro.

There will be time for that.

# Functions and Arguments

## Required Arguments

Arguments appearing before special markers such as `&optional` and `&rest` are required.

For:

```lisp
(defun add (a b)
  (+ a b))
```

both arguments must be supplied:

```lisp
(add 10 20)
```

## Optional Arguments

Optional arguments follow `&optional`:

```lisp
(defun greeting (name &optional title)
  (if title
      (message "Good morning, %s %s!" title name)
    (message "Hello, %s!" name)))
```

We can now call:

```lisp
(greeting "John")
```

or:

```lisp
(greeting "John" "Mr.")
```

If `title` is omitted, its value is `nil`.

So:

```lisp
(if title ...)
```

neatly distinguishes the two cases.

## Any Number of Arguments

`&rest` means:

> collect all remaining arguments into a list.

For example:

```lisp
(defun sum-everything (first &rest rest)
  "Add FIRST to all remaining numbers."
  (let ((sum first))
    (dolist (number rest)
      (setq sum (+ sum number)))
    (message "Sum: %d" sum)))
```

Calling:

```lisp
(sum-everything 10 1 2 3)
```

makes:

```lisp
rest
```

contain:

```lisp
(1 2 3)
```

# Interactive Calls

We already know that `interactive` allows a function to become an Emacs command.

It can also specify where its arguments should come from.

A few particularly useful codes are:

```text
s   text entered by the user
n   number
f   name of an existing file
F   file name; the file need not exist yet
b   name of an existing buffer
B   buffer name; the buffer need not exist yet
r   beginning and end of the region as two numeric arguments
```

This is nowhere near the complete list.

It is long enough to write some genuinely useful commands already.

And short enough not to regret learning Lisp just yet.

# Configuring Emacs

All right.

What was all this knowledge for?

To stop blindly copying:

```lisp
(setq ...)
(add-hook ...)
(require ...)
```

from the Internet.

Or at least to copy it with slightly greater awareness.

## Where Does the Configuration Live?

Historically, the classic file was:

```text
~/.emacs
```

and Emacs still supports it.

There are, however, more convenient locations:

```text
~/.emacs.d/init.el
```

and the XDG-compatible:

```text
~/.config/emacs/init.el
```

Modern Emacs supports both.

There is one important trap: the traditional locations take precedence.

Emacs checks locations including:

```text
~/.emacs.el
~/.emacs
~/.emacs.d/init.el
```

and if one of them exists, it may never reach:

```text
~/.config/emacs/init.el
```

For this series I will use:

```text
~/.emacs.d/init.el
```

It is a long-established, familiar layout and remains convenient once the configuration begins to spread across several files.

If an old:

```text
~/.emacs
```

exists and the destination `init.el` does not, it can be moved:

```bash
mkdir -p ~/.emacs.d
mv ~/.emacs ~/.emacs.d/init.el
```

# A Separate File for Customize

In the previous episode we met **Customize**.

Settings saved through it are ultimately written as Emacs Lisp code.

Your `init.el` may contain something like:

```lisp
(custom-set-faces
 ;; custom-set-faces was added by Custom.
 ;; If you edit it by hand, you could mess it up, so be careful.
 ;; Your init file should contain only one such instance.
 ;; If there is more than one, they won't work right.
 '(default
   ((t
     (:inherit nil
      :extend nil
      :stipple nil
      :inverse-video nil
      :box nil
      :strike-through nil
      :overline nil
      :underline nil
      :slant normal
      :weight regular
      :height 143
      :width normal
      :foundry "SRC"
      :family "Hack Nerd Font Mono")))))
```

I would rather not mix machine-generated customization with configuration written by hand.

So let us create a separate file:

```text
~/.emacs.d/custom.el
```

If existing `custom-set-variables` or `custom-set-faces` forms generated by Customize already live in `init.el`, they can be moved there.

Then add this to `init.el`:

```lisp
;; Keep Customize-generated settings in a separate file.
(setq custom-file
      (expand-file-name "custom.el" user-emacs-directory))

;; Load it if it already exists.
(when (file-exists-p custom-file)
  (load custom-file))
```

And now, instead of treating the code as an incantation, we can actually read it.

## `user-emacs-directory`

The variable:

```lisp
user-emacs-directory
```

points to the directory used for the user's Emacs files.

With our layout, that is:

```text
~/.emacs.d/
```

We can check it:

```lisp
(message "%s" user-emacs-directory)
```

With another configuration layout, its value may be different.

## `expand-file-name`

This expression:

```lisp
(expand-file-name "custom.el" user-emacs-directory)
```

constructs the full path to the file inside the Emacs user directory.

We could hard-code something like:

```text
/home/user/.emacs.d/custom.el
```

but then the configuration immediately becomes less portable.

It may eventually move to another computer, another account, another init directory, or even an operating system with strange opinions about path names.

Best not to assume too much.

## `setq custom-file`

This line:

```lisp
(setq custom-file ...)
```

tells Customize:

> from now on, write your generated settings here.

## `load`

Finally:

```lisp
(load custom-file)
```

loads the contents of `custom.el`.

Telling Customize where to save the code is not enough.

On the next startup, that code still needs to be executed.

## Why `file-exists-p`?

If the file does not exist yet:

```lisp
(load custom-file)
```

signals an error.

So we use:

```lisp
(when (file-exists-p custom-file)
  (load custom-file))
```

In other words:

> if the file exists, load it.

And we have just written our first piece of configuration that, several pages ago, would have looked like a mysterious collection of parentheses.

Progress.

# Summary

These really are only the absolute basics of Emacs Lisp.

We have not even touched:

- macros,
- anonymous functions,
- hooks,
- keymaps,
- symbols and their properties,
- `require` and `provide`,
- packages,
- more advanced data structures,
- or any of the subjects that cause the number of parentheses to begin growing exponentially.

Good.

The point was simply to make those cryptic snippets pasted into the One True Editor's configuration file a little less cryptic from now on.

When we encounter:

```lisp
(when (file-exists-p custom-file)
  (load custom-file))
```

we no longer need to take the Internet entirely on faith.

We can at least say:

> Ah. I know what those parentheses are doing.

More or less.
