# VTE OSC 52 packaging

Packaging for [VTE with OSC 52 clipboard support](https://github.com/asifamin13/vte), intended for opt-in use by VTE-based terminal applications without replacing system VTE.

## Status

Preliminary repository: packaging recipes, CI workflows, and a launcher are not implemented yet.

Planned scope:

- RPM packaging first, with Debian packaging later.
- Installation under `/opt/vte-osc52`, separate from system packages.
- A `vte-osc52-run` launcher that sets library and typelib paths per application rather than globally.
- CI builds and verification, including isolated OSC 52 clipboard testing.

The initial target is GTK 3 applications using `Vte-2.91`, such as Terminator. Applications using GTK 4 require a corresponding VTE build; terminals that do not use VTE cannot use this library.

## Source

The VTE fork is included as a submodule at `subs/vte`, initially pinned to tag `0.84.1-osc52.1`. Git records the exact source commit in the parent repository.

After cloning this repository, initialize the source checkout:

```sh
git submodule update --init --recursive
```

## Clipboard security

OSC 52 clipboard writes allow terminal output, including output from remote programs, to replace the local clipboard. Clipboard reads can expose local clipboard contents to those programs. The intended configuration supports writes without enabling reads.
