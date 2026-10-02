# Version Range Translator

Paste one version range and see how npm, Cargo, Composer, PEP 440 and RubyGems each read it, as plain comparators tested against versions you list.

**Live demo:** https://0xelitesystem.github.io/version-range-translator/

## The receipt

The same string is read correctly, and differently, by each ecosystem's own grammar:

- `~1.2` stops below 1.3 in npm and Cargo, but below 2.0 in Composer. npm prints `>=1.2.0 <1.3.0-0`, the Cargo Book gives `~1.2 := >=1.2.0, <1.3.0`, and composer/semver parses it to `[>= 1.2.0.0-dev < 2.0.0.0-dev]`. Composer's own `versions.md` says `~1.2` is equivalent to `>=1.2 <2.0.0`.
- `~>1.2` stops below 1.3 in npm, but below 2.0 in RubyGems. RubyGems defines `~>` as `v >= r && v.release < r.bump` in `Gem::Requirement::OPS`, and its upstream test asserts that `1.5` satisfies `~> 1.4` while `2.0` does not.
- `~=1.2` stops below 1.3 in npm, but below 2.0 in PEP 440. packaging's `SpecifierSet("~=1.2")` contains `1.9` and not `2.0`.
- A bare `1.2` is 1.2.x in npm, below 2.0.0 in Cargo (a bare version is a caret requirement), exactly 1.2.0 in Composer and RubyGems, and not a valid specifier at all in PEP 440.

Why does npm accept `~>` and `~=`? node-semver's `internal/re.js` defines its tilde token as `(?:~>?)`, and the version that follows may start with any run of `v`, `=` and whitespace. So `~>1.2` and `~=1.2` both reach npm as its own tilde. The README's range grammar lists only `tilde ::= '~' partial`. Neither side is wrong. Each follows its own rules, and the page shows which rules are in play.

## Use

1. Open the live demo, or open `index.html` from a local clone.
2. Type one range in **Version range**. Type test versions in the box beside it, one per line.
3. Read the **Where the answers differ** strip first. It lists every test version that gets a different answer across the columns that accept the range.
4. Read the **Five readings** cards. Each shows VALID or REFUSED (with the reason in plain words and the engine's own message), the engine's native reading, and the plain comparators it reduces to.
5. Read the **Test versions** table. For every line it shows how each ecosystem parses the literal text, whether it matches the requirement, and whether that ecosystem's default policy would admit it, with the reason.
6. Use **Equivalent spelling** to rewrite the range for another ecosystem. Pick the source. If the source reading is one interval, the page emits `>=A <B` in each target's syntax, re-reads every emitted string with the target's engine, and prints how many of your test versions it agrees on. The agreement count is the only claim. Unions (`||`), `!=` and PEP 440 `===` are refused.

Test versions are never pre-translated. `1.3.0-rc.1` is `1.3.0rc1` to PEP 440, `1.3.0.pre.rc.1` to RubyGems and `1.3.0.0-RC1` to Composer, while npm and Cargo read it as SemVer. `1.2` is not a version in npm or Cargo. `v1.2.3` is a version in npm, Composer and PEP 440, and not in Cargo or RubyGems. Each engine gets the line exactly as you typed it.

Real situations this is for:

- **Porting a manifest.** Moving a dependency spec from `package.json` to `Cargo.toml`, or from a gemspec to `pyproject.toml`. Paste the old range, choose its ecosystem as the source, and copy the target spelling that agrees on the versions you care about.
- **Reading an advisory.** A security advisory states an affected range in one ecosystem's syntax. Paste it, list the versions you actually run, and see which ones fall inside under each grammar before you trust a copied range.
- **Reviewing a pull request.** Someone writes `~>` in a `package.json` or `~1.2` in a `composer.json` because that is what it means where they came from. Paste it and the cards show what this ecosystem will actually do.
- **Pre-release surprises.** Check whether a release candidate would be picked up. The table separates "matches the requirement" from "admitted by default", because those are decided at different layers in different ecosystems.
- **PEP 440 candidate sets.** Whether pip-style resolution keeps a pre-release depends on what else is available. List the candidates and tick or untick `--pre` to see the filter result.

## What each column runs

| Column | Ported from | What is modelled |
|---|---|---|
| npm | node-semver 7.8.5 (`classes/range.js`, `comparator.js`, `semver.js`, `internal/re.js`) | Strict mode. Pre-release matching per the README's "Prerelease Tags" section. Default install policy equals the match. |
| Cargo | semver crate 1.0.28 (`src/parse.rs`, `src/eval.rs`), the crate Cargo uses for version requirements | Default operator is caret, commas required, at most 32 comparators, a wildcard must stand alone, no `v` prefix, leading zeros refused. Pre-release rule per `matches_req` and `pre_is_compatible`. |
| Composer | composer/semver 3.5.0 (`VersionParser`, `Constraint`, `MultiConstraint`) plus PHP's `version_compare` from php-src `ext/standard/versioning.c` | Constraint matching ignores stability. The "admitted" row applies `minimum-stability`, which defaults to `stable`, explicit `@` stability flags, and the flag Composer infers for a root requirement (composer/composer 2.10.3 `RootPackageLoader::extractStabilityFlags`): a single-token constraint that names a pre-release, such as `^1.2.3-beta.2`, admits that stability and above. The range is read as your own `composer.json` require. |
| PEP 440 | pypa/packaging 26.3 (`Version`, `Specifier`, `SpecifierSet`) | Full version parsing and normalization, all eight operators, prefix matching, local labels, the exclusive comparison rules, and `SpecifierSet.filter` over your test list as the candidate set. Already-installed pre-releases are not modelled. |
| RubyGems | RubyGems v4.0.22 (`Gem::Version`, `Gem::Requirement`, `Gem::Dependency#match?`) | The range is split on commas into separate requirement strings, the way a Gemfile or gemspec passes them. `satisfied_by?` is the match row, `Dependency#match?` is the admitted row. |

Every port is pure string logic and runs offline.

RubyGems' `VERSION_PATTERN` uses an atomic group, which JavaScript lacks. A plain group accepts the same strings there, because the token after it never starts with a letter or digit. composer/semver uses PCRE possessive quantifiers, which JavaScript also lacks. Those are emulated with a capturing lookahead plus a backreference, and the port is checked against real PHP output rather than by reading.

## How it was verified

The expected values inside the page were produced by real engines, then frozen into `index.html`:

- **npm, Composer, PEP 440:** generated by running node-semver 7.8.5, composer/semver 3.5.0 under PHP 8.5.6, and packaging 26.3 under Python 3.14 over a shared list of ranges and versions. That list includes the 18-range matrix `~1.2`, `1.2.3`, `1.2`, `=1.2`, `~>1.2`, `~=1.2`, `^1.2.3`, `^0.2.3`, `>=1.2`, `^1.2.3-beta.2`, `>=1.2, <1.5`, `>=1.2 <1.5`, `>=1.2 || <1.0`, `1.2 - 2`, `~1`, `~=1`, `~> 1` and `1.*`.
- **Cargo:** taken from `tests/test_version_req.rs` at tag 1.0.28. The same matrix was also run through the semver 1.0.28 crate compiled from source.
- **Composer inferred stability flags:** generated by running `extractStabilityFlags` from composer/composer 2.10.3, copied into a PHP script with composer/semver 3.5.0, over 19 root constraints, including `^1.2.3-beta.2` (beta), `>=1.2-beta <2` (beta, because the AND split leaves `>=1.2-beta` as one token), `1.3.0-RC1` (RC) and the hyphen range `1.0-beta - 2.0` (none).
- **RubyGems:** taken from `test/rubygems/test_gem_requirement.rb` and `test_gem_version.rb` at tag v4.0.22. The same matrix was also run through the v4.0.22 library files under Ruby.

The page runs every vector on load and prints `N of N vectors pass`, computed live. If any vector fails, the failing row is shown in red.

Three rows from the matrix worth seeing in the page: Composer `^1.2.3-beta.2` matches `1.2.4-rc.1` while npm and Cargo do not. Cargo `^1.2.3-beta.2` matches `1.2.3-beta.5`. npm `>=1.2 || <1.0` matches `0.9.0` and `1.3.0` and not `1.1.0`.

Upstream versions were re-checked at build time against each project's latest release. composer/semver 3.4.4 and 3.5.0 gave identical results on the vector set, as did packaging 26.2 and 26.3. node-semver 7.7.4 (the copy bundled with npm on the build machine) matched 7.8.5 everywhere except the malformed x-range `1.x.3`. RubyGems' three library files are byte-identical between v4.0.20 and v4.0.22.

## Sources

- Cargo Book, specifying dependencies (caret, tilde and wildcard tables, pre-releases): https://doc.rust-lang.org/cargo/reference/specifying-dependencies.html
- Composer, versions and constraints (hyphen, wildcard, tilde, caret, stability constraints): https://github.com/composer/composer/blob/main/doc/articles/versions.md
- Composer, schema (`minimum-stability` defaults to `stable`, stability flags): https://github.com/composer/composer/blob/main/doc/04-schema.md
- PyPA version specifiers (compatible release, exclusive ordered comparison, handling of pre-releases): https://packaging.python.org/en/latest/specifications/version-specifiers/
- node-semver README (advanced range syntax, pre-release tags, range grammar): https://github.com/npm/node-semver/blob/main/README.md
- RubyGems `requirement.rb`, `version.rb` and `dependency.rb` at v4.0.22: https://github.com/rubygems/rubygems/tree/v4.0.22/lib/rubygems
- semver crate source and tests at 1.0.28: https://github.com/dtolnay/semver/tree/1.0.28
- composer/semver source: https://github.com/composer/semver
- composer/composer `RootPackageLoader::extractStabilityFlags` (inferred stability flags for root requirements) at 2.10.3: https://github.com/composer/composer/blob/2.10.3/src/Composer/Package/Loader/RootPackageLoader.php
- pypa/packaging source: https://github.com/pypa/packaging
- php-src `version_compare`: https://github.com/php/php-src/blob/master/ext/standard/versioning.c

## Why this exists

A version range is a contract, and copying one between ecosystems copies the characters, not the contract. The failure is quiet: nothing errors, the resolver simply picks a different set of versions than the person who wrote the range intended. Single-ecosystem documentation answers "what does this mean here". This page answers "what does this exact string mean in each of the five places it might end up", and lets you test the answer on your own versions.

It is one HTML file with no dependencies, no build step, no telemetry and no network access of any kind. MIT licensed. Fork it, vendor it, keep it behind your firewall.

This is an independent project. It is not affiliated with, endorsed by, or sponsored by npm, the Rust project, Composer, the Python Packaging Authority (PyPA) or RubyGems. Their names appear only to say whose grammar each column ports.

## Privacy

Everything runs in your browser. Nothing is uploaded, logged or sent anywhere. There is no analytics, no font fetch, no CDN reference and no `fetch` call in the source, and the page's Content Security Policy blocks every connection. Load it once and it works offline, including from `file://`. The only thing written to storage is your light or dark theme preference.

## Run locally

```
git clone https://github.com/0xelitesystem/version-range-translator.git
cd version-range-translator
```

Open `index.html` directly in a browser, or serve it:

```
python -m http.server 8000
```

Then visit http://localhost:8000/.

## Build

There is no build. It is a single self-contained `index.html` with inline CSS and JavaScript, and the test vectors are inline data in the same file.

## Related

- [semantic-versioning-reference](https://github.com/0xelitesystem/semantic-versioning-reference) explains the SemVer convention in plain language. This tool shows what five resolvers actually do with a range.
- [dependency-supply-chain-reference](https://github.com/0xelitesystem/dependency-supply-chain-reference) covers why the version you resolve matters for supply-chain risk.
- [yaml-type-coercion-auditor](https://github.com/0xelitesystem/yaml-type-coercion-auditor) is the same idea for YAML scalars: one input, several parsers, every difference shown.

## Third-party notices

No third-party code is bundled. Each column's behaviour was reimplemented in JavaScript from reading that project's source. No code is reproduced; the patterns are rewritten to accept the same strings, and correctness is checked against the vectors below.

- **npm/node-semver.** Copyright (c) Isaac Z. Schlueter and Contributors. ISC License. https://github.com/npm/node-semver
  Taken: behaviour reimplemented in JavaScript from reading the source; no code is reproduced; expected values generated by running node-semver 7.8.5.
- **dtolnay/semver.** Its `LICENSE-MIT` file carries no copyright line. Licensed MIT or Apache-2.0. https://github.com/dtolnay/semver
  Taken: behaviour reimplemented in JavaScript from reading the source; no code is reproduced; test vectors taken from `tests/test_version_req.rs` at tag 1.0.28.
- **composer/semver.** Copyright (C) 2015 Composer. MIT License. https://github.com/composer/semver
  Taken: behaviour reimplemented in JavaScript from reading the source; no code is reproduced; expected values generated by running composer/semver 3.5.0 under PHP.
- **composer/composer.** Copyright (c) Nils Adermann, Jordi Boggiano. MIT License. https://github.com/composer/composer
  Taken: the stability-flag inference in `RootPackageLoader::extractStabilityFlags` reimplemented in JavaScript from reading the source; no code is reproduced; expected values generated by running that function under PHP.
- **pypa/packaging.** Copyright (c) Donald Stufft and individual contributors. BSD license (`LICENSE.BSD`; the project is offered under either that or Apache-2.0). https://github.com/pypa/packaging
  Taken: behaviour reimplemented in JavaScript from reading the source; no code is reproduced; expected values generated by running packaging 26.3.
- **rubygems/rubygems.** "RubyGems is copyrighted free software by Chad Fowler, Rich Kilmer, Jim Weirich and others." MIT License or the Ruby terms (`LICENSE.txt`). https://github.com/rubygems/rubygems
  Taken: behaviour reimplemented in JavaScript from reading the source; no code is reproduced; test vectors taken from `test/rubygems/test_gem_requirement.rb` and `test/rubygems/test_gem_version.rb` at tag v4.0.22.
- **php/php-src, `ext/standard/versioning.c`.** Copyright © The PHP Group and Contributors. BSD-3-Clause (SPDX identifier in the file header). https://github.com/php/php-src
  Taken: behaviour of `version_compare` reimplemented in JavaScript from reading the source; no code is reproduced; expected values generated by running PHP 8.5.6.

## License

MIT. Copyright (c) 2026 0xelitesystem.
