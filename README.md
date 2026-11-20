# BoringSpeedDials

A speed-dial new-tab page for Chrome and Edge: a grid of website tiles, organized into groups, that asks for as few permissions as possible.

**Status:** specification stage. No code yet. See [speed-dial-spec.md](speed-dial-spec.md).

## Goals

- **No permission warnings at install.** The only install-time permission is `storage`, which shows no warning; it's used only to read policies set by administrators. Everything else is optional and requested only when you turn on the feature that needs it.
- **Local first.** The first release works entirely offline, with no server at all.
- **End-to-end encrypted sync (later).** The server can't read your dials. A default server will be provided, and you can host your own.
- **No analytics or telemetry.**

## License

[MIT](LICENSE)
