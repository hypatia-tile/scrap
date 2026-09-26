# Why a Flymake backend is not running

Flymake merges any number of backends, each listed in the buffer-local
`flymake-diagnostic-functions`. When diagnostics do not appear, three things
can be true, and none of them announces itself clearly.

Everything below was checked on Emacs 30.2 unless marked otherwise.

## A backend that signals is disabled, and is not retried

A backend is expected to return quickly or signal. If it signals, Flymake does
not merely skip it for that check — it puts it on the buffer's disabled list
and never calls it again. From its own docstring (`flymake.el`):

> backend functions are expected to return quickly or signal an error, in
> which case the backend is disabled. Flymake will not try disabled backends
> again for any future checks of this buffer. To reset the list of disabled
> backends, turn `flymake-mode` off and on again, or interactively call
> `flymake-start` with a prefix argument.

The code:

```elisp
((and (not force)
      (flymake--state-disabled state))
 (flymake-log :debug "Backend %s is disabled, not starting" ...))
```

The consequence that costs something: **fix the cause in a live session and
nothing changes.** `M-x flymake-start` will not call the backend, because the
argument that overrides the disabled list is `FORCE`, the second one. Use
`(flymake-start nil t)`, `C-u M-x flymake-start`, or toggle `flymake-mode`.

And `flymake-start` returns `nil` on success, so its return value is no signal
at all:

```
$ emacs --batch --eval '(progn (require (quote flymake))
    (with-temp-buffer (emacs-lisp-mode) (insert "(defun f () 1)")
      (message "%S" (flymake-start))))'
nil
```

## `Flymake:!` in the mode line means exactly one thing

The `!` is not a generic error marker. It is one branch of
`flymake--mode-line-exception`:

```elisp
(cond ((zerop (hash-table-count flymake--state))
       '("?" nil "No known backends"))
      ((cl-set-difference running reported)
       `("Wait" ... "Waiting for %s running backend(s)"))
      ((and (flymake-disabled-backends) (null running))
       '("!" compilation-mode-line-run
         "All backends disabled"))
      (t '(nil nil nil)))
```

So `?` is "no backends registered", `!` is "backends registered, all
disabled". The indicator is clickable and runs
`flymake-switch-to-log-buffer`, which holds the explanation each backend was
disabled with — the first place to look, and easy to walk past.

## eglot replaces the backend list rather than adding to it

An extra backend added from a major-mode hook disappears when a language
server takes over the buffer. eglot does not `add-hook`; it assigns
(`eglot.el`, `eglot--managed-mode`):

```elisp
(eglot--setq-saving flymake-diagnostic-functions '(eglot-flymake-backend))
```

Measured in a TSX buffer with a hand-written backend on the major-mode hook:

| | `flymake-diagnostic-functions` |
| --- | --- |
| after the major-mode hook | `(my-backend t)` |
| once eglot was managing the buffer | `(eglot-flymake-backend)` |

`eglot-stay-out-of` is not the way around it. That assignment is the only place
eglot installs its own backend — grepping `eglot.el` for
`flymake-diagnostic-functions` finds one site — so telling eglot to stay out of
the variable removes the LSP diagnostics along with the clobber. (Read from
source; not run.)

What works instead is `eglot-managed-mode-hook`, which runs after the
assignment, adding the backend back buffer-locally. Note that eglot has already
called `(flymake-mode 1)` by then and the first check has run without the
re-added backend, so it needs its own `flymake-start` to report on the buffer
as opened.

## Emacs 31: every backend sits behind `trusted-content`

Emacs 30.1 introduced `trusted-content` for `elisp-flymake-byte-compile`,
which *evaluates* what it checks — byte compilation expands macros. Emacs 31
extends the check to every backend, and an undeclared buffer does not get a
skipped backend but a **disabled** one, with the consequences above:

```
Disabling eglot-flymake-backend in FILE (untrusted content)
```

and `Flymake:!`, and no diagnostics whatsoever — including a language
server's type errors, which have nothing to do with evaluating the buffer.

*This section is reported from an Emacs 31.1 machine, not measured here:*
Emacs 30.2's `flymake.el` contains neither `trusted-content` nor
`flymake-always-safe`, so the gate cannot be observed on 30.

The gate takes either of two keys:

```elisp
(if (or (trusted-content-p) (function-get backend 'flymake-always-safe))
    (apply backend ...)
  (user-error "Disabling %S in %s (untrusted content)" backend (buffer-name)))
```

Putting the property on a single backend is the narrower grant, and for a
language server's backend it gives away nothing new: it does not evaluate the
buffer, it forwards what the server produced, and the server was started on
that file without consulting `trusted-content` at all.

```elisp
(function-put 'eglot-flymake-backend 'flymake-always-safe t)
```

Confirmed on the 31.1 machine: with the property set and the backend forced off
the disabled list, the type errors appear. Declaring directories in
`trusted-content` instead is the broader promise — it also lets
`elisp-flymake-byte-compile` evaluate code in everything under them, which
matters if those trees hold repositories cloned from other people.

Set at init time the ordering problem disappears: the property is in place
before any backend runs, so nothing is ever disabled and no `FORCE` is needed.

## Debugging order

1. `Flymake:?` or `Flymake:!` in the mode line — registered at all, or
   disabled?
2. `M-x flymake-switch-to-log-buffer` for the reason.
3. `flymake-diagnostic-functions` in that buffer — is the backend still listed,
   or did something assign over it?
4. Only then re-run, and re-run with `FORCE`.
