# How lsp_go works

This repository is a Loom language profile for `gopls`. It holds no
code. It holds one `extension.toml` that tells [Loom](https://github.com/Roasbeef/loom)
how to run `gopls` in its jail, a small Go module in `fixture/` that the
server loads, and a CI workflow that proves the two agree. This document
walks through each of them so you can read `extension.toml` without
having Loom's source open.

For the machinery behind it, see Loom's
[language-server architecture](https://github.com/Roasbeef/loom/blob/main/docs/architecture/lsp.md)
and [ADR-016, language profiles](https://github.com/Roasbeef/loom/blob/main/docs/adr/016-language-profiles.md).
The short version: Loom speaks the Language Server Protocol and knows no
language. A server runs project code (`gopls` runs `go list`), so Loom
runs it inside a sandbox, and everything that sandbox must grant, plus
the few facts a language spells differently, comes from a profile like
this one.

## Reading path and design rules

This is the only design document in the repository, because the profile
is one short TOML file and a split into principles and architecture would
repeat itself. Read `extension.toml` first, then `fixture/`, then
`.github/workflows/check.yml`; this document explains each in that order.
Three rules shaped the file.

### Grant what was measured, and nothing wider

The jail starts from nothing, so every root in the table is there
because something failed without it. `gopls` writes nothing into the
module, so the project stays read-only. The module cache is the only
host directory granted, and read-only. The reason is that a language
server runs project code, and a grant that is wider than the measurement
is authority a hostile project can use.

### Keep every cache the host trusts out of the jail

A directory the host's own tools read without checking must never be one
the jailed server can write. `go build` trusts `GOCACHE`, so the jailed
`gopls` gets a build cache of its own, under a directory Loom owns,
instead of `~/.cache/go-build`. This is the one place the profile spends
more keys than a plain grant would, and the reason is the exfiltration
path a shared cache opens.

### Put the proof beside the claim

A profile asserts that a server loads a project and answers inside the
jail. The `[[check]]`s and the fixture are that assertion made
executable, and CI runs them against a real Loom, so a claim in
`extension.toml` that stops being true fails a build instead of a
user's session.

## extension.toml, key by key

### `[extension]`

`name = "lsp_go"` is what `loomd ext check` takes, and `tier =
"profile"` says this extension is data only. A profile may declare
language-server tables and checks. It may not declare tools, hooks or a
network policy, and Loom refuses the manifest if it does. `version`,
`description` and `license` are ordinary metadata.

### `[lsp.go]`

The table name, `go`, is the server's name inside Loom. A `loom.toml`
table with the same name replaces this one whole, never field by field.

`command = ["gopls"]` is the argv Loom executes. It is never a shell
string. A bare name is looked up on the daemon's `PATH`, so `gopls` must
be on it (`go install` puts it in `~/go/bin`, which is on few). Loom
mounts the directory holding the executable, read-only, and nothing
wider.

`extensions = [".go"]` claims every `.go` file for this server. A file
has exactly one owning server, so two profiles claiming `.go` conflict
and Loom refuses both. Because `language_id` is not set, documents are
opened with the `languageId` `go`, the default (the first extension
without its dot), which is what `gopls` expects.

`root_markers = ["go.mod"]` chooses the project: the nearest ancestor
directory of a file that holds a `go.mod` is the module root, and it is
the root `gopls` is started on. A file whose real location lies outside
that root is refused before any request is sent.

`project` is not set, so it defaults to `"read-only"`. `gopls` writes
nothing into the module. (Gleam's server is the contrast: it writes
`build/`, and its profile says `"writable"`.)

`readable = ["~/go/pkg/mod"]` grants the module cache read-only. A module
with requirements has their sources there and `go list` reads them. The
fixture has no requirements, so its checks pass without this line, but
any real module needs it. If `go env GOMODCACHE` prints somewhere else,
copy the table into `loom.toml` and name that path.

`cache_env = { XDG_CACHE_HOME = "xdg", GOCACHE = "go-build", GOPLSCACHE =
"gopls" }` sets three environment variables to three directories that
Loom owns, under `<cache>/loom/lsp/go/`. Loom creates each one and grants
it writable. Each exists for a reason:

- `go list` must be able to write a build cache, or it loads no packages
  and every question comes back not found.
- That cache must not be the host's own `~/.cache/go-build`. The host's
  `go build` trusts `GOCACHE` without verifying it, and `go list` in the
  jail runs cgo with the project's `#cgo` flags, which the model can
  write. A shared cache would let the jailed server plant entries the
  host later links.
- `GOCACHE` and `GOPLSCACHE` are named instead of left to follow
  `XDG_CACHE_HOME`, because Go and `gopls` read `XDG_CACHE_HOME` on Linux
  only. On macOS their defaults are under `~/Library/Caches`, which the
  jail does not grant. Naming them keeps both caches private on every
  platform.
- `XDG_CACHE_HOME` stays for the other tools `gopls` runs that read it,
  such as the `goimports` cache.
- The three directories are siblings, because Loom refuses a cache inside
  another: the server could swap the inner one for a link. They outlive a
  session, so only the first query on a host starts cold.

Measured with this fixture on Linux: both checks pass, `gopls` writes
into the private `gopls` directory, and the host's `~/.cache/go-build`
is untouched.

`env = ["GOFLAGS"]` passes one variable through from the daemon's
environment, by name only. The value never lives in the file. It carries
an operator's build tags and `-mod` mode, without which `gopls` would load
a different package set than their build does.

`hint` is one line, appended once to `lsp_definition`'s description in
sessions that configure this server. It tells the model to write
`util.Greet`, which is how Loom's resolver splits a qualified name here.
`qualifier_separators` and `module_case` are not set: the defaults, `.`
and as-written, already fit Go. The qualifier `util` must end the
definition's directory (`util/`), which it does.

What the table does not say matters too. There is no writable root, and
the network is off in every jail. The Go installation itself (`GOROOT`)
must be readable in the jail, which holds for `/usr`, `/opt` and
Homebrew's prefix and not for a toolchain cache elsewhere; the README
covers that.

### `[[check]]`

A check is one question asked of a running server through the same door
Loom's tools use, and a list of the sites the answer must equal. Sites
are fixture-relative `path:line`, compared as a set, so two hits on one
line count once and order does not matter. This profile has two:

- **`definition` of `util.Greet`, expecting `util/util.go:4`.** It proves
  that `gopls` starts under the jail's policy, that the caches work (or
  no package would load), that Loom's bare-name search finds the
  candidate sites, and that the qualifier `util` narrows them to the
  right directory.
- **`references` of `util.Greet`, expecting `util/util.go:4`,
  `main.go:6` and `main.go:7`.** That is the declaration, two calls on one
  line of `main.go`, and one call on the next. It proves the server
  loaded both packages and answers across the package boundary.

The checks do not prove the `readable` grant (the fixture has no
requirements), `GOFLAGS`, or that your `GOROOT` is readable.

## The fixture

`fixture/` is a Go module, `example.com/fixture`, with two packages.
`util/util.go` declares `Greet` on line 4, and `main.go` imports `util`
and calls `Greet` on lines 6 (twice) and 7. It has no dependencies, so
`go list` needs no network, and the jail has none. The checks assert line
numbers, so editing a fixture file means updating `expect`.

## What the CI does

`.github/workflows/check.yml` runs on pushes to `main`, pull requests and
manual dispatch. Its single job proves the profile against a real Loom:

1. **Check out two repositories.** This one goes into `profile/` and Loom
   into `loom/`, at the revision `LOOM_REV`, so the tree that is later
   installed is exactly this repository.
2. **Install the build tools.** Erlang/OTP, Gleam and rebar3 build and
   run `loomd`, and Go builds the sandbox helper and is also the language
   toolchain here.
3. **Prepare the runner for the sandbox.** Install `bubblewrap` (the
   namespace and mount work) and `ripgrep` (what a bare-name question
   searches with). Lift Ubuntu 24.04's AppArmor restriction on
   unprivileged user namespaces, and delegate a cgroup v2 base to the
   helper with probes that prove the delegation is real.
4. **Build Loom.** `make -C loom sandbox` builds the helper, and `make -C
   loom server-shipment` builds `loomd`, retrying Hex fetches.
5. **Install `gopls` v0.23.0**, the version Loom's own jail lane pins and
   the one these checks were measured against.
6. **Install the profile and check it.** `loomd ext install` on the
   `profile/` checkout, then `loomd ext check lsp_go`. The job passes
   exactly when that command exits 0.

### Re-proving against a newer Loom

`LOOM_REV` in the workflow's `env` block is the Loom revision the profile
is proven against. At the time of writing it is a branch name, because
Loom's language-server work had not merged; the comment beside it says to
repin it to a commit SHA on Loom's `main` once it has. To re-prove the
profile against a newer Loom, change that one value, push, and read the
`check lsp_go` job. A failure prints a `FAIL` line naming the expected
and the actual sites. To do the same by hand, build `loomd` from that
Loom and run the last step's two commands.
