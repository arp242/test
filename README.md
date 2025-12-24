
Files with variation selectors and other "invisible" characters can appear as
something they're not; there are a bunch of files in this repo:

    "main.go"
    "not_a_t{\uFE0E}est.go"
    "not_a_t{\u200E}est2.go"
    "not_a_t{\u200D}est3.go"
    "not_compiled.go{\uFE04}"
    "not_compiled2{\u2024}go"

This is a problem especially for languages that operate on directories and/or
have assumptions about filenames; I'll use Go here as an example, but much of
this probably applies to at least some other languages as well.

Go typically operates on directories ("packages") rather than individual files.
It considers anything ending with `.go` to be a Go source file to compile.
However, `not_compiled.go` isn't compiled because it ends with `go{\uFE04}`. The
`not_compiled2.go` file has "ONE DOT LEADER" instead of a regular full stop (a
somewhat different but similar issue).

Go considers any file ending with `_test.go` to be a test file. This won't get
compiled when you run `go build`, but run on `go test`. However, the variation
selector, zero-width space, or zero-width joiner in `test` prevents these files
from being recognized as a test file, and it will be compiled in to the program
instead.

Especially the `not_a_test` files are rather insidious; the `not_compiled.go`
files don't have syntax highlights and there is at least *some* clue something
might be amiss, and it's probably also harder to do something nefarious by not
compiling code (although I wouldn't rule that out!), but the `not_a_test.go`
file seems fine, and allow injecting code. Running this with Go produces:

    % go run .
    main.go
    not_a_test2.go
    not_a_test3.go
    not_a_test.go

This gives an opportunity to "hide" code in plain sight. A simple example:

    // main.go:
    var bcryptCost = bcrypt.DefaultCost

    func hashPassword(pwd []byte) []byte {
        pwd, _ := bcrypt.GenerateFromPassword(pwd, bcryptCost)
        reurn pwd
    }

    // not_a_test.go:
    func init() {
        // Lower bcrypt cost in tests, because otherwise any test will take
        // well over a second as it's so slow.
        bcryptCost = bcrypt.MinCost
    }

Now all passwords are hashed less secure, but this is hidden in such a way that
makes it appear fine.

There are probably other opportunities for skulduggery here; but this kind of
stuff is the most obvious. Hiding logs, disabling rate-limiters, disabling (or
enabling) feature gates are other obvious ones.

---

All of this is made worse because many tools don't display this at all. I tested
a bunch of software, and the only cases where it displays clearly are:

- CLI git (`git log --stat`, `git ls-files`, etc.), which shows the UTF-8 byte
  sequences.

- zsh autocomplete, which shows `<codepoint>` when *inserting* (not when
  listing).

- URL bar (*not* URL preview!) when viewing the files in web-based git browsers.
  For example: https://github.com/arp242/test/blob/master/not_a_t%EF%B8%8Eest.go

Other than that, I couldn't find any way to display this other than really
looking for it (`echo * | hexdump -C`).

Vim, VSCode, macOS finder, Windows explorer, /bin/ls – they all just display
`not_a_test.go`. Arguably, that's how it should be, at least for some of these
programs. Variation selectors are a Unicode feature and *shouldn't* display in
normal text. However, filenames are not really "normal text", certainly not in
the context of code, as this can easily be abused in nefarious ways. This is
similar to using LTR tricky to hide code (e.g. https://github.com/nickboucher/trojan-source).
