# MIT License

Copyright (c) 2026 The ALDL ECU Simulator authors

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.

---

## Third-party dependencies

The following Rust crates are used under their own permissive licences
(MIT or Apache-2.0, both compatible with this project's MIT licence):

| Crate | Licence | Purpose |
|-------|---------|---------|
| [serialport](https://crates.io/crates/serialport) | MPL-2.0 | Serial-port bindings |
| [clap](https://crates.io/crates/clap) | MIT / Apache-2.0 | Command-line argument parser |
| [tracing](https://crates.io/crates/tracing) | MIT | Application-level tracing |
| [tracing-subscriber](https://crates.io/crates/tracing-subscriber) | MIT | tracing subscriber |
| [anyhow](https://crates.io/crates/anyhow) | MIT / Apache-2.0 | Error handling |
| [eframe](https://crates.io/crates/eframe) | MIT / Apache-2.0 | egui GUI framework (GUI build only) |
| [rfd](https://crates.io/crates/rfd) | MIT | File dialogs (GUI build only) |

The full dependency tree with per-crate licence info is available via
`cargo metadata --format-version 1` at the project root.

## External dependencies

**No external kernel drivers, virtual COM port software, or other
non-Rust dependencies are bundled.** The simulator connects to real
COM ports only (typically a USB-to-serial adapter + null-modem cable
on a bench rig). The `build.sh` cross-compile target is
`x86_64-pc-windows-gnu`; the resulting `.exe` files are
self-contained.
