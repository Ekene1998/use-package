# `use-package`

[![Join the chat at https://gitter.im/use-package/Lobby](https://badges.gitter.im/use-package/Lobby.svg)](https://gitter.im/use-package/Lobby?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)
[![Build Status](https://github.com/jwiegley/use-package/actions/workflows/test.yml/badge.svg)](https://github.com/jwiegley/use-package/actions)
[![GNU ELPA](https://elpa.gnu.org/packages/use-package.svg)](https://elpa.gnu.org/packages/use-package.html)

The `use-package` macro allows you to isolate package configuration in your
`.emacs` file in a way that is both performance-oriented and, well, tidy.  I
created it because I have over 80 packages that I use in Emacs, and things
were getting difficult to manage.  Yet with this utility my total load time is
around 2 seconds, with no loss of functionality!

**NOTE**: `use-package` is **not** a package manager! Although `use-package`
does have the useful capability to interface with package managers (see
[below](#package-installation)), its primary purpose is for the configuration
and loading of packages.

- [Installing use-package](#installing-use-package)
- [Getting started](#getting-started)
- [Key-binding](#key-binding)
	+ [Binding to keymaps](#binding-to-keymaps)
	+ [Binding within local keymaps](#binding-within-local-keymaps)
- [Modes and interpreters](#modes-and-interpreters)
- [Magic handlers](#magic-handlers)
- [Hooks](#hooks)
- [Package customization](#package-customization)
  + [Customizing variables](#customizing-variables)
  + [Customizing faces](#customizing-faces)
- [Notes about lazy loading](#notes-about-lazy-loading)
- [Information about package loads](#information-about-package-loads)
- [Conditional loading](#conditional-loading)
	+ [Conditional loading before :preface](#conditional-loading-before-preface)
	+ [Loading packages in a sequence](#loading-packages-in-sequence)
	+ [Prevent loading if dependencies are missing](#prevent-loading-if-dependencies-are-missing)
- [Byte compiling your .emacs](#byte-compiling-your-emacs)
	+ [Prevent a package from loading at compile-time](#prevent-a-package-from-loading-at-compile-time)
- [Extending the load-path](#extending-the-load-path)
- [Catching errors during use-package expansion](#catching-errors-during-use-package-expansion)
- [Diminishing and delighting minor modes](#diminishing-and-delighting-minor-modes)
- [Package installation](#package-installation)
	+ [Usage with other package managers](#usage-with-other-package-managers)
- [Gathering Statistics](#gathering-statistics)
- [Keyword Extensions](#keyword-extensions)
	+ [use-package-ensure-system-package](#use-package-ensure-system-package)
	+ [use-package-chords](#use-package-chords)
	+ [How to create an extension](#how-to-create-an-extension)
		+ [First step: Add the keyword](#first-step-add-the-keyword)
		+ [Second step: Create a normalizer](#second-step-create-a-normalizer)
		+ [Third step: Create a handler](#third-step-create-a-handler)
		+ [Fourth step: Test it out](#fourth-step-test-it-out)
- [Some timing results](#some-timing-results)
* [Upgrading to 2.x](#upgrading-to-2x)
	+ [Semantics of :init is now consistent](#semantics-of-init-is-now-consistent)
	+ [:idle has been removed](#idle-has-been-removed)
	+ [:defer now accepts an optional numeric argument](#defer-now-accepts-an-optional-numeric-argument)
	+ [Add :preface, occurring before everything except :disabled](#add-preface-occurring-before-everything-except-disabled)
	+ [Add :functions, for declaring functions to the byte-compiler](#add-functions-for-declaring-functions-to-the-byte-compiler)
	+ [use-package.el is no longer needed at runtime](#use-packageel-is-no-longer-needed-at-runtime)
## Installing use-package

Either clone from this GitHub repository or install from
[GNU ELPA](https://elpa.gnu.org/) (recommended).

## Getting started

Here is the simplest `use-package` declaration:

``` elisp
;; This is only needed once, near the top of the file
(eval-when-compile
  ;; Following line is not needed if use-package.el is in ~/.emacs.d
  (add-to-list 'load-path "<path where use-package is installed>")
  (require 'use-package))

(use-package foo)
```

This loads in the package `foo`, but only if `foo` is available on your
system. If not, a warning is logged to the `*Messages*` buffer.

Use the `:init` keyword to execute code before a package is loaded.  It
accepts one or more forms, up to the next keyword:

``` elisp
(use-package foo
  :init
  (setq foo-variable t))
```

Similarly, `:config` can be used to execute code after a package is loaded.
In cases where loading is done lazily (see more about autoloading below), this
execution is deferred until after the autoload occurs:

``` elisp
(use-package foo
  :init
  (setq foo-variable t)
  :config
  (foo-mode 1))
```

As you might expect, you can use `:init` and `:config` together:

``` elisp
(use-package color-moccur
  :commands (isearch-moccur isearch-all)
  :bind (("M-s O" . moccur)
         :map isearch-mode-map
         ("M-o" . isearch-moccur)
         ("M-O" . isearch-moccur-all))
  :init
  (setq isearch-lazy-highlight t)
  :config
  (use-package moccur-edit))
```

In this case, I want to autoload the commands `isearch-moccur` and
`isearch-all` from `color-moccur.el`, and bind keys both at the global level
and within the `isearch-mode-map` (see next section).  When the package is
actually loaded (by using one of these commands), `moccur-edit` is also
loaded, to allow editing of the `moccur` buffer.

If you autoload non-interactive function, please use `:autoload`.

```elisp
(use-package org-crypt
  :autoload org-crypt-use-before-save-magic)
```

## Key-binding

Another common thing to do when loading a module is to bind a key to primary
commands within that module:

``` elisp
(use-package ace-jump-mode
  :bind ("C-." . ace-jump-mode))
```

This does two things: first, it creates an autoload for the `ace-jump-mode`
command and defers loading of `ace-jump-mode` until you actually use it.
Second, it binds the key `C-.` to that command.  After loading, you can use
`M-x describe-personal-keybindings` to see all such keybindings you've set
throughout your `.emacs` file.

A more literal way to do the exact same thing is:

``` elisp
(use-package ace-jump-mode
  :commands ace-jump-mode
  :init
  (bind-key "C-." 'ace-jump-mode))
```

When you use the `:commands` keyword, it creates autoloads for those commands
and defers loading of the module until they are used.  Since the `:init` form
is always run -- even if `ace-jump-mode` might not be on your system --
remember to restrict `:init` code to only what would succeed either way.

The `:bind` keyword takes either a cons or a list of conses:

``` elisp
(use-package hi-lock
  :bind (("M-o l" . highlight-lines-matching-regexp)
         ("M-o r" . highlight-regexp)
         ("M-o w" . highlight-phrase)))
```

Alternatively, the command name may be replaced with a cons `(desc . command)`,
where `desc` is a string describing `command`, which is the name of a command
to bind to:

```elisp
(use-package avy
  :bind ("C-:" ("Jump to char" . avy-goto-char)
         "M-g f" ("Jump to line" . avy-goto-line)))
```

These descriptions can be used by other code that deals with key bindings.
For example, the GNU ELPA package which-key displays them when showing key
bindings, instead of the plain command names.

The `:commands` keyword takes either a symbol or a list of symbols.

**NOTE**: inside strings, special keys like `tab` or `F1`-`Fn` have to be written inside angle brackets, e.g. `"C-<up>"`.
Standalone special keys (and some combinations) can be written in square brackets, e.g. `[tab]` instead of `"<tab>"`. The syntax for the keybindings is similar to
the "kbd" syntax: see [https://www.gnu.org/software/emacs/manual/html_node/emacs/Init-Rebinding.html](https://www.gnu.org/software/emacs/manual/html_node/emacs/Init-Rebinding.html)
for more information.

Examples:

``` elisp
(use-package helm
  :bind (("M-x" . helm-M-x)
         ("M-<f5>" . helm-find-files)
         ([f10] . helm-buffers-list)
         ([S-f10] . helm-recentf)))
```

Furthermore, [remapping commands](https://www.gnu.org/software/emacs/manual/html_node/elisp/Remapping-Commands.html)
with `:bind` and `bind-key` works as expected, because when the
binding is a vector, it is passed straight to `define-key`. So the
following example will rebind `M-q` (originally `fill-paragraph`) to
`unfill-toggle`:

``` elisp
(use-package unfill
  :bind ([remap fill-paragraph] . unfill-toggle))
```

### Binding to keymaps

Normally `:bind` expects that commands are functions that will be autoloaded
from the given package. However, this does not work if one of those commands
is actually a keymap, since keymaps are not functions, and cannot be
autoloaded using Emacs' `autoload` mechanism.

To handle this case, `use-package` offers a special, limited variant of
`:bind` called `:bind-keymap`. The only difference is that the "commands"
bound to by `:bind-keymap` must be keymaps defined in the package, rather than
command functions. This is handled behind the scenes by generating custom code
that loads the package containing the keymap, and then re-executes your
keypress after the first load, to reinterpret that keypress as a prefix key.

For example:

``` elisp
(use-package projectile
  :bind-keymap
  ("C-c p" . projectile-command-map))
```

### Binding within local keymaps

Slightly different from binding a key to a keymap, is binding a key *within* a
local keymap that only exists after the package is loaded.  `use-package`
supports this with a `:map` modifier, taking the local keymap to bind to:

``` elisp
(use-package helm
  :bind (:map helm-command-map
         ("C-c h" . helm-execute-persistent-action)))
```

The effect of this statement is to wait until `helm` has loaded, and then to
bind the key `C-c h` to `helm-execute-persistent-action` within Helm's local
keymap, `helm-command-map`.

Multiple uses of `:map` may be specified. Any binding occurring before the
first use of `:map` are applied to the global keymap:

``` elisp
(use-package term
  :bind (("C-c t" . term)
         :map term-mode-map
         ("M-p" . term-send-up)
         ("M-n" . term-send-down)
         :map term-raw-map
         ("M-o" . other-window)
         ("M-p" . term-send-up)
         ("M-n" . term-send-down)))
```
### Binding to repeat-maps

A special case of binding within a local keymap is when that keymap is
used by `repeat-mode`. These keymaps are usually defined specifically
for this. Using the `:repeat-map` keyword, and passing it a name for
the map it defines, will bind all following keys inside that map, and
(by default) set the `repeat-map` property of each bound command to
that map.


This creates a keymap called `git-gutter+-repeat-map`, makes four
bindings in it as above, then sets the `repeat-map` property of each
bound command (`git-gutter+-next-hunk` `git-gutter+-previous-hunk`,
`git-gutter+-stage-hunks` and `git-gutter+-revert-hunk`) to that
keymap.

``` elisp
(use-package git-gutter+
  :bind
  (:repeat-map git-gutter+-repeat-map
   ("n" . git-gutter+-next-hunk)
   ("p" . git-gutter+-previous-hunk)
   ("s" . git-gutter+-stage-hunks)
   ("r" . git-gutter+-revert-hunk)))
```

Specifying `:exit` inside the scope of `:repeat-map` will prevent the
`repeat-map` property being set, so that the command can be used from
within the repeat map, but after it using it the repeat map will no
longer be available. This is useful for commands often used at the end
of a series of repeated commands:

``` elisp
(use-package git-gutter+
  :bind
  (:repeat-map my/git-gutter+-repeat-map
   ("n" . git-gutter+-next-hunk)
   ("p" . git-gutter+-previous-hunk)
   ("s" . git-gutter+-stage-hunks)
   ("r" . git-gutter+-revert-hunk)
   :exit
   ("c" . magit-commit-create)
   ("C" . magit-commit)
   ("b" . magit-blame)))
```

Specifying `:continue` *forces* setting the `repeat-map` property
(just like *not* specifying `:exit`), so these two snippets are
equivalent:

``` elisp
(use-package git-gutter+
  :bind
  (:repeat-map my/git-gutter+-repeat-map
   ("n" . git-gutter+-next-hunk)
   ("p" . git-gutter+-previous-hunk)
   ("s" . git-gutter+-stage-hunks)
   ("r" . git-gutter+-revert-hunk)
   :exit
   ("c" . magit-commit-create)
   ("C" . magit-commit)
   ("b" . magit-blame)))
```

``` elisp
(use-package git-gutter+
  :bind
  (:repeat-map my/git-gutter+-repeat-map
   :exit
   ("c" . magit-commit-create)
   ("C" . magit-commit)
   ("b" . magit-blame)
   :continue
   ("n" . git-gutter+-next-hunk)
   ("p" . git-gutter+-previous-hunk)
   ("s" . git-gutter+-stage-hunks)
   ("r" . git-gutter+-revert-hunk)))
```
   



## Modes and interpreters

Similar to `:bind`, you can use `:mode` and `:interpreter` to establish a
deferred binding within the `auto-mode-alist` and `interpreter-mode-alist`
variables. The specifier to either keyword can be a cons cell, a list of cons
cells, or a string or regexp:

``` elisp
(use-package ruby-mode
  :mode "\\.rb\\'"
  :interpreter "ruby")

;; The package is "python" but the mode is "python-mode":
(use-package python
  :mode ("\\.py\\'" . python-mode)
  :interpreter ("python" . python-mode))
```

If you aren't using `:commands`, `:bind`, `:bind*`, `:bind-keymap`,
`:bind-keymap*`, `:mode`, `:interpreter`, or `:hook` (all of which imply `:defer`; see
the docstring for `use-package` for a brief description of each), you can
still defer loading with the `:defer` keyword:

``` elisp
(use-package ace-jump-mode
  :defer t
  :init
  (autoload 'ace-jump-mode "ace-jump-mode" nil t)
  (bind-key "C-." 'ace-jump-mode))
```

This does exactly the same thing as the following:

``` elisp
(use-package ace-jump-mode
  :bind ("C-." . ace-jump-mode))
```

## Magic handlers

Similar to `:mode` and `:interpreter`, you can also use `:magic` and
`:magic-fallback` to cause certain function to be run if the beginning of a
file matches a given regular expression. The difference between the two is
that `:magic-fallback` has a lower priority than `:mode`. For example:

``` elisp
(use-package pdf-tools
  :load-path "site-lisp/pdf-tools/lisp"
  :magic ("%PDF" . pdf-view-mode)
  :config
  (pdf-tools-install :no-query))
```

This registers an autoloaded command for `pdf-view-mode`, defers loading of
`pdf-tools`, and runs `pdf-view-mode` if the beginning of a buffer matches the
string `"%PDF"`.

## Hooks

The `:hook` keyword allows adding functions onto package hooks. The
following are equivalent:

``` elisp
(use-package company
  :hook (prog-mode . company-mode))

(use-package company
  :commands company-mode
  :init
  (add-hook 'prog-mode-hook #'company-mode))
```

And likewise, when multiple hooks should be applied, all of the
following are also equivalent:

``` elisp
(use-package company
  :hook ((prog-mode text-mode) . company-mode))

(use-package company
  :hook ((prog-mode . company-mode)
         (text-mode . company-mode)))

(use-package company
  :hook (prog-mode . company-mode)
  :hook (text-mode . company-mode))

(use-package company
  :hook
  (prog-mode . company-mode)
  (text-mode . company-mode))

(use-package company
  :commands company-mode
  :init
  (add-hook 'prog-mode-hook #'company-mode)
  (add-hook 'text-mode-hook #'company-mode))
```

When using `:hook`, omit the "-hook" suffix if you specify the hook
explicitly, as this is appended by default. For example, the following
code will not work as it attempts to add to the `prog-mode-hook-hook`
which does not exist:

``` elisp
;; DOES NOT WORK
(use-package ace-jump-mode
  :hook (prog-mode-hook . ace-jump-mode))
```

If you do not like this behaviour, set `use-package-hook-name-suffix`
to nil. By default the value of this variable is "-hook".

The use of `:hook`, as with `:bind`, `:mode`, `:interpreter`, etc., causes the
functions being hooked to implicitly be read as `:commands` (meaning they will
establish interactive `autoload` definitions for that module, if not already
defined as functions), and so `:defer t` is also implied by `:hook`.

## Package customization

### Customizing variables.

The `:custom` keyword allows customization of package custom variables.

``` elisp
(use-package comint
  :custom
  (comint-buffer-maximum-size 20000 "Increase comint buffer size.")
  (comint-prompt-read-only t "Make the prompt read only."))
```

The documentation string is not mandatory.

**NOTE**: these are only for people who wish to keep customizations with their
accompanying use-package declarations. Functionally, the only benefit over
using `setq` in a `:config` block is that customizations might execute code
when values are assigned.

**NOTE**: The customized values are **not** saved in the Emacs `custom-file`.
Thus you should either use the `:custom` option **or** you should use `M-x
customize-option` which will save customized values in the Emacs `custom-file`.
Do not use both.

### Customizing faces

The `:custom-face` keyword allows customization of package custom faces.

``` elisp
(use-package eruby-mode
  :custom-face
  (eruby-standard-face ((t (:slant italic)))))

(use-package example
  :custom-face
  (example-1-face ((t (:foreground "LightPink"))))
  (example-2-face ((t (:foreground "LightGreen"))) face-defspec-spec))

(use-package zenburn-theme
  :preface
  (setq my/zenburn-colors-alist
        '((fg . "#DCDCCC") (bg . "#1C1C1C") (cyan . "#93E0E3")))
  :custom-face
  (region ((t (:background ,(alist-get my/zenburn-colors-alist 'cyan)))))
  :config
  (load-theme 'zenburn t))
```

## Notes about lazy loading

The keywords `:commands`, et al, provide "triggers" that cause a package to be
loaded when certain events occur. However, if `use-package` cannot determine
that any trigger has been declared, it will load the package immediately (when
Emacs is starting up) unless `:defer t` is given. The presence of triggers can
be overridden using `:demand t` to force immediately loading anyway. For
example, `:hook` represents a trigger that fires when the specified hook is
run.

In almost all cases you don't need to manually specify `:defer t`, because
this is implied whenever `:bind` or `:mode` or `:interpreter` are used.
Typically, you only need to specify `:defer` if you know for a fact that some
other package will do something to cause your package to load at the
appropriate time, and thus you would like to defer loading even though
`use-package` has not created any autoloads for you.

You can override package deferral with the `:demand` keyword. Thus, even if
you use `:bind`, adding `:demand` will force loading to occur immediately and
not establish an autoload for the bound key.

## Information about package loads

When a package is loaded, and if you have `use-package-verbose` set to `t`, or
if the package takes longer than 0.1s to load, you will see a message to
indicate this loading activity in the `*Messages*` buffer.  The same will
happen for configuration, or `:config` blocks that take longer than 0.1s to
execute.  In general, you should keep `:init` forms as simple and quick as
possible, and put as much as you can get away with into the `:config` block.
This way, deferred loading can help your Emacs to start as quickly as
possible.

Additionally, if an error occurs while initializing or configuring a package,
this will not stop your Emacs from loading.  Rather, the error will be
captured by `use-package`, and reported to a special `*Warnings*` popup
buffer, so that you can debug the situation in an otherwise functional Emacs.

## Conditional loading

You can use the `:if` keyword to predicate the loading and initialization of
modules.

For example, I only want `edit-server` running for my main,
graphical Emacs, not for other Emacsen I may start at the command line:

``` elisp
(use-package edit-server
  :if window-system
  :init
  (add-hook 'after-init-hook 'server-start t)
  (add-hook 'after-init-hook 'edit-server-start t))
```
In another example, we can load things conditional on the operating system:

``` elisp
(