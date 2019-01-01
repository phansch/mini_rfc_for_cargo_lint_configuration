# Mini-RFC: Cargo Lint configuration

This is a rough plan for adding lint configuration by config file to Cargo.

It's a continuation of [this Cargo issue][that_issue] from @rantanen. Relevant
parts were copied and used as a starting ground for this text. @detrumi opened a
[Cargo PR][detrumi_pr] when the issue was opened. However, it was decided at the
time that the design should go through more discussion first. That's what this
Mini-RFC is meant to kick-off.

## Summary

Rust currently only allows configuring lints on the command line or via crate
level attributes. This Mini-RFC proposes an additional way to configure lint
levels and possibly lint-specific settings in `Cargo.toml`.

## Motivation

Configuring lint levels on the command line or via crate level attributes is
fine when the project consists of a single crate, but gets more cumbersome when
the project is a workspace with more than a few crates.

Being able to define the lints in an external file that will be used when
building the crates would have several benefits:

 * Sharing one configuration file for all the crates in a workspace, therefore
   ensuring consistent lint levels everywhere.
 * A canonical place to check for lint configuration when contributing to a new
   project.
 * An easy way to examine the version history for lint changes.

Some real world examples of where this could bring improvements:

* [This rustc rollup][rollup] which denies a single lint in various subcrates
  over multiple PRs.
* [serde's lib.rs][ex_serde] and [serde_derive's lib.rs][ex_serde2] would also
  benefit from having a central lint configuration file. The same for
  [diesel's lib.rs][ex_diesel] and [diesel_migrations's lib.rs][ex_diesel2].
* The [Amethyst game framework][amethyst], with 15 different sub crates, has to
  enable warnings for the `rust_2018_ideoms` and `rust_2018_compatibilty` lint
  groups in [every][am_1] [single][am_2] [one][am_3] [of][am_4] [the][am_5]
  [sub-crates][am_6] (that's 6 of them linked here).
* Similarly, [ripgrep][ripgrep] denies the `missing_docs` lint in all of its 9
  workspace members: [GitHub search][ripgrep_search]

Previously this topic came up a few times in the Clippy issue tracker as well:

* [#574: Ability to conditionally set levels for lints when not using clippy through cargo features](previous_574)
* [#1313: Allow/deny lints in clippy.toml](previous_1313)
* [#3164: It is impossible to understand from the readme file how to suppress lints using clippy.toml](previous_3164)

## Guide-level explanation

Lint configuration can also be done through a configuration file.  The
configuration needs to be able to configure the `allow/warn/deny/forbid` lint
levels for each lint. In order to be more teachable, using toml syntax is
probably the best way to configure the lints. The most obvious location is in
the `Cargo.toml` file.

## Reference-level explanation

In the most basic version, it would be possible to set the
`allow/warn/deny/forbid` of lints that are registered with the lint registry.
These would be grouped in a new `[lints]` section. Tool-specific lints (like
Clippy) are grouped together using the tool name.

That would give us a format like this:

```toml
[lints]
dead_code = "allow"

[lints.clippy]
non_snake_case = "allow"
```

This format would make git history easy to read and would allow you to add
configuration options to lints later on. It also allows grouping of lints on
more than just the lint level.

And if Cargo/rustc ever support lint configurations, this would be more future proof:

```toml
[lints.clippy]
cyclomatic_complexity = { state = "allow", threshold = 30 }
```

Another possible format would be:

```toml
[lints]
allow = ["dead_code", "non_snake_case"]
```

This has the benefit of not having to repeat the lint level for every single
lint. However, this would probably make diffs more difficult to read.

### `cargo check` and tool lints interaction

If a user has configured a tool lint in `Cargo.toml` and runs `cargo check`, the
implementation needs to make sure to only pass the rustc lints to `rustc`,
otherwise rustc would complain about unknown lints because it was invoked with
`check` and not `clippy`, for example.

When `cargo clippy` is executed, it should include the rustc lints, as it does
today.

### Lint precedence

The lints can be specified on the workspace level and for individual packages.
Anything on the package level will override the workspace setup on a per lint
basis. This means that lints configured in the workspace will act as a default
for packages within that workspace, and can be overridden by individual packages.

### Why Cargo.toml?

The biggest advantage of using `Cargo.toml` is that it's already used for
configuring project and workspace level settings. Most users would expect to
find lint configuration there and pretty much every Rust project uses it
already.

However, one downside is that there is currently no way to share a `Cargo.toml`
between separate projects. There's no concrete solution here, but maybe
a new `inherit_from` or `inherit_lints_from` key could solve that problem. Prior
art includes at least [Rubocop's `inherit_from` setting][rubocop_inherit] and
[eslint's `extend` setting][eslint_inherit].

Additionally, one might argue that `Cargo.toml` is part of Cargo, which should
just be concerned about package management and not lint configuration. However,
Cargo is currently the only tool available that interfaces with `rustc` and
offers a way to configure interfacing with rustc.

For users it may also not be immediately clear that both `cargo check` and
`cargo clippy` work with the same underlying lint system and they may not expect
to be able to configure tool lints through a Cargo configuration file. This is
because other programming language ecosystems usually have completely separate
tools for their lints. See the _Prior art and Rust_ section below for a small
discussion of this problem.

If we find a solution to `Cargo.toml` not being shareable across projects, we
consider it to be the best approach mainly because the file is already present
in every Cargo project.

## Alternatives

### Using `package.metadata` tables

Another approach, that was [proposed on IRLO recently][other_irlo_post], could
be using the `metadata` configuration:

Cargo would look for a `[package.metadata.lints]` section instead of a `[lints]`
section. Tools that are not hooking into the lint system, could use their own
`[package.metadata.toolname]` section instead of using a custom file.

One problem with `[package.metadata]` is that it does not support [virtual
workspaces][virtual_workspaces] as it requires the `package` table to be
present in the `Cargo.toml`.

### Different configuration file location

There are some other plausible locations to configure lints, such as
`.cargo/config` or `Lints.toml`.

#### .cargo/config

A more standard location, though less well-known, would be `.cargo/config` (see [Cargo reference](cargo_docs)).
As with `Cargo.toml`, this file would need a new `[lints]` section, too.

One big advantage would be that users could add their personal lint preferences
in their home directory (both `$HOME/.cargo/config` and `.cargo/config` are
supported) as `.cargo/config` already
[has a hierarchical configuration lookup][cargo_hierarchical].

A downside would be that Lint configuration via config file would supposedly be
pretty common, while custom Cargo configuration is rarely used. Almost every
project would want to make use of a lint configuration, which means that every
project would end up having to create the additional `.cargo/config` file.

#### Lints.toml

Another option would be a completely separate file. Maybe called `Lints.toml`.
This would be the most flexible implementation because we would not have to care
about existing code for `Cargo.toml` or `.cargo/config`. It is also more similar
to the existing [`Clippy.toml`][clippy_toml].

However, it would also mean that, like with `.cargo/config`, users have to add
an additional configuration file to the roots of their repositories.

Additionally, we may also want to handle `rustfmt` configuration, and we would
need to find a more general name.

## Prior art

In other programming language ecosystems, the concerns of dependency management
and things such as lint configuration are handled by completely separate tools.
This is usually because the language itself does not come with any lints like
Rust. For example, in Javascript, you have [eslint][eslint] and the
`package.json`, which don't really interact. In Ruby, you have
[Rubocop][rubocop] for lints and `bundler`/`Gemfile` for dependencies.

Rust is different from these examples because it already comes with built-in
lints and offers an interface for external tools to make use of the same lint
system.

If another language exists that provides external tools with hooks into its lint
system, it would be good to take a look.

## Future possibilities

* It might be desirable to also be able to configure the settings of specifics lints, such as in the `cyclomatic_complexity` shown earlier. This could also replace
the [`clippy.toml`][clippy_toml] file, which currently allows configuring lints.
* It would make sense to include other settings related to lints, such as output verbosity or `--cap-lints` in the lint configuration.
* Other tools could also add lints to the same lints section, not just Clippy.

### Diagnostics for lint level definition location reporting

With the current approach, we pass all lint level definitions from the config
file to rustc via the command line arguments. Consequently, rustc will tell the
user the lint level has been defined on the command-line. This is the opposite
of helpful if the user wants to change the level of the lint:

```text
error: any use of this value will cause an error
  --> $DIR/const-err-multi.rs:13:1
   |
LL | pub const A: i8 = -std::i8::MIN;
   | ^^^^^^^^^^^^^^^^^^-------------^
   |                   |
   |                   attempt to negate with overflow
   |
   = note: requested on the command line with `-D const_err`
```

Ideally, the user would get a `note` like this:

```text
note: lint level defined here
  --> $DIR/Cargo.toml:511:1
   |
LL | const_err = { state = "deny" }
   | ^^^^^^^^^
```

Adding a new variant to [`LintSource`][lintsource] is the easy part, but how is this
information passed from Cargo to rustc? Should we pass a json file with lint
level definition locations to rustc everytime cargo invokes rustc?

## TODO text

Things that still need work or aren't even included in the text, yet:

1. Difference between 'workspace level' and 'individual packages'?
1. In general expand the Pros/Cons of the configuration file section
1. How much of an issue is errors for unknown lints? My feeling is that it
   should be OK, but it does set a floor for the minimum supported rustc for
   something that is not really critical.

## Unresolved Questions

1. There is currently no good way to detect whether a lint is known or not: all configured lints are passed to rustc directly, which throws
an error if the lint isn't known. Ideally, Cargo would issue a warning instead.

## Before publishing on IRLO

1. Look for TODOs in text
1. Ensure all github code links are permalinks
1. Ask in #wg-clippy if anyone wants to cross-check it

[that_issue]: https://github.com/rust-lang/cargo/issues/5034
[ex_serde]: https://github.com/serde-rs/serde/blob/5c24f0f0f300c7bd21bad5b097f6f1919de8477c/serde/src/lib.rs#L87-L134
[ex_serde2]: https://github.com/serde-rs/serde/blob/5c24f0f0f300c7bd21bad5b097f6f1919de8477c/serde_derive/src/lib.rs#L18-L49
[ex_diesel]: https://github.com/diesel-rs/diesel/blob/59aa49b65713df8d666991b37f5e18011f3671d5/diesel/src/lib.rs#L132-L170
[ex_diesel2]: https://github.com/diesel-rs/diesel/blob/36078014717d6c2fb0d03d2a10d19177c06ed86d/diesel_migrations/src/lib.rs#L1-L21
[rollup]: https://github.com/rust-lang/rust/pull/52268
[eslint]: https://eslint.org/docs/user-guide/getting-started#configuration
[rubocop]: https://docs.rubocop.org/en/latest/basic_usage/
[amethyst]: https://github.com/amethyst/amethyst
[am_1]: https://github.com/amethyst/amethyst/blob/8e7f06a9bca8b60664855cac12dd74be7ddc0c82/amethyst_derive/src/lib.rs#L2
[am_2]: https://github.com/amethyst/amethyst/blob/8e7f06a9bca8b60664855cac12dd74be7ddc0c82/amethyst_animation/src/lib.rs#L48
[am_3]: https://github.com/amethyst/amethyst/blob/8e7f06a9bca8b60664855cac12dd74be7ddc0c82/amethyst_assets/src/lib.rs#L10
[am_4]: https://github.com/amethyst/amethyst/blob/8e7f06a9bca8b60664855cac12dd74be7ddc0c82/amethyst_audio/src/lib.rs#L1
[am_5]: https://github.com/amethyst/amethyst/blob/8e7f06a9bca8b60664855cac12dd74be7ddc0c82/amethyst_config/src/lib.rs#L6
[am_6]: https://github.com/amethyst/amethyst/blob/8e7f06a9bca8b60664855cac12dd74be7ddc0c82/amethyst_controls/src/lib.rs#L3
[ripgrep]: https://github.com/BurntSushi/ripgrep
[ripgrep_search]: https://github.com/BurntSushi/ripgrep/search?q=missing_docs&unscoped_q=missing_docs
[detrumi_pr]: https://github.com/rust-lang/cargo/pull/5728
[previous_574]: https://github.com/rust-lang/rust-clippy/issues/574
[previous_1313]: https://github.com/rust-lang/rust-clippy/issues/1313
[previous_3164]: https://github.com/rust-lang/rust-clippy/issues/3164
[cargo_config]: https://doc.rust-lang.org/cargo/reference/config.html
[other_irlo_post]: https://internals.rust-lang.org/t/tool-configs-in-cargo-toml-particularily-rustfmt-clippy/9055
[lintsource]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc/lint/enum.LintSource.html
[rubocop_inherit]: https://docs.rubocop.org/en/latest/configuration/#inheriting-from-another-configuration-file-in-the-project
[eslint_inherit]: https://eslint.org/docs/developer-guide/shareable-configs
[clippy_toml]: https://github.com/rust-lang/rust-clippy#configuration
[cargo_hierarchical]: https://doc.rust-lang.org/cargo/reference/config.html#hierarchical-structure
[virtual_workspaces]: https://doc.rust-lang.org/cargo/reference/manifest.html#virtual-manifest
