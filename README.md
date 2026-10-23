![Nonius](img/banner.png)

# Nonius

Dense time series, from server to pixel, exact to the nanosecond at any
zoom.

This repository is Nonius's product page, served at
<https://andrelaf.github.io/nonius-site/>, and its public
specifications:

- [`specs/protocol.md`](specs/protocol.md): the wire protocol, a
  history in tiles over HTTP and its images, written to be implemented
  without Nonius's code;
- [`specs/pyramid-file.md`](specs/pyramid-file.md): a series and its
  pyramid of summaries in one file, read in place.

Nonius's Rust crates are on [crates.io](https://crates.io/crates/nonius-core)
and its browser package on [npm](https://www.npmjs.com/package/nonius); the
.NET packages are coming to NuGet.

MIT or Apache-2.0, at your option.
