# Mini-RFC: Cargo Lint configuration

This is a rough plan for adding lint configuration by config file to Cargo.

It's a continuation of [this Cargo issue][that_issue]. I copied over relevant parts
that were written by @rantanen and expanded on additional parts. @detrumi opened
a [PR for Cargo][detrumi_pr] when the issue was opened. However, it was decided
at the time that the design should go through more discussion first. That's what
this Mini-RFC is meant to kick-off.

## Summary

Configure lint levels and possibly lint specific settings via a config file.

## Motivation

Currently project-wide lint configuration needs to be included in the crate
sources. This is okay when the project consists of a single crate, but gets
more cumbersome when the project is a workspace with dozen crates.

Being able to define the lints in an external file that will be used when
building the crates would have several benefits:

 * Sharing one configuration file for all the crates in a workspace therefore
   ensuring consistent lint levels everywhere.
 * A canonical place to check for lint configuration when contributing to a new
   project.
 * An easy way to examine the version history for lint changes.

Some real world examples of where this could bring improvements:

* [This rustc rollup][rollup] which denies a single lint in various subcrates
  over multiple PRs.
* [serde's lib.rs][ex_serde] and [serde_derive's lib.rs][ex_serde2] would also
  benefit from having a central lint configuration file. The same for the
  [diesel lib.rs][ex_diesel] and [diesel_migrations lib.rs][ex_diesel2].
* The [Amethyst game framework][amethyst] with 15 different sub crates has to
  enable warnings for the `rust_2018_ideoms` and `rust_2018_compatibilty` lint
  groups in [every][am_1] [single][am_2] [one][am_3] [of][am_4] [the][am_5]
  [sub-crates][am_6] (that's 6 of them linked here).
* Similarily [ripgrep][ripgrep] denies the `missing_docs` lint in all of its 9
  workspace members: [GitHub search][ripgrep_search]

Previously this topic came up a few times in the Clippy issue tracker as well:

* https://github.com/rust-lang/rust-clippy/issues/574
* https://github.com/rust-lang/rust-clippy/issues/1313
* https://github.com/rust-lang/rust-clippy/issues/3164

## Prior art and Rust

In other programming language ecosystems the concerns of dependency management
and things such as lint configuration are handled by completely separate tools.
This is usually because the language itself does not come with any lints like
Rust. For example, in Javascript you have [eslint][eslint] and the package.json
which don't really interact. In Ruby you have [Rubocop][rubocop] for lints and
then bundler for dependencies.

Rust is different from these examples because it already comes with built-in
lints and offers external tools to make use of the same lint system.

If another language exists that provides external tools with hooks into its lint
system, it would be good to take a look.

## Guide-level explanation

Lint configuration can also be done through a configuration file.  The
configuration needs to be able to configure the `allow/warn/deny/forbid` lint
levels for each lint. In order to be more teachable, using toml syntax is
probably the best way to configure the lints. 


## Reference-level explanation

In the most basic version, it would be possible to set the
`allow/warn/deny/forbid` of lints that are registered with the lint registry.

That would give us a format like this:

```toml
dead_code = "allow"
non_snake_case = "allow"
```

This format would make git history easy to read and would allow adding lint
configuration options later on. It also allows to group lints together in other
ways than just the lint level.

Another possible format would be:

```toml
allow = [dead_code, non_snake_case, ...]
```

This has the benefit of not having to repeat the lint level for every single
lint. However, this woul probably make diffs more difficult to read.

### Lint precedence

The lints can be specified on the workspace level and for individual packages.
Anything on the package level will override the workspace setup on per lint
basis. Specifying clippy-lints will result in Clippy complaining about unknown
lints if Clippy isn't used.

Also if Cargo/rustc ever support lint configurations, this would be more future proof:

```toml
cyclomatic_complexity = { state = "allow", threshold = 30 }
```

The next section discusses the different places where the lint configuration
could be stored. There are upsides and downsides for each of the locations.

### Which configuration file?

#### Cargo.toml

The `Cargo.toml` is already used for configuring project and workspace level
settings. I suppose most users would expect to find lint configuration in here. 

Using the `Cargo.toml` to configure lints would need a new `[lints]` section.


Rust is different compared to other languages because we have the built-in lints
(via `cargo check`) and then tool lints which use the same mechanism but through
a different command line interface. For users it may not be immedeately clear that
both commands work with the same underlying system. Therefore, new users may not
expect to be able to configure tool lints through a Cargo configuration file.

- Pro: Everyone already has it. No need for additional files.
- Con: People might say that the Cargo.toml is part of Cargo which should just be concerned about package management and not lint configuration.
- Con: Users might not expect external tool configuration to go into Cargo.toml

* Is Cargo.toml really the best place? Need to clarify why to choose one approach over the others.
  * If the use case is for large organizations to share common settings, then this assumes they have a monolithic workspace which may not be true for everyone.
  * The counter argument for something like lints.toml is the proliferation of many config files for a Cargo project (rustfmt.toml, lints.toml, .cargo/config, etc.).

#### .config/cargo

As with `Cargo.toml` this file would need a new `[lints]` section, too. 

- Pro: users could add their personal lint preferences in their home directory.
- Con: I would estimate that almost every project would want to make use of a lint configuration file, which means that every project would end up having to create the additional `.config/cargo` file.
(Both rubocop and eslint support this)


#### Lints.toml

A third option would be a completely separate file. Maybe called `Lints.toml`.
This would be the most flexible implementation because we would not have to care
about existing code for `Cargo.toml` or `.config/cargo`.

However, it would also mean that, like with `.config/cargo`, users have to add
an additional configuration file to the roots of their repositories.

Additionally, we may also want to handle `rustfmt` configuration and we would
need to find a more general name. See the next section on other tool
configuration.

TODO: Summarize the different config files and come to a conclusion

## Alternatives

* Is there a more general solution? How does this relate to other tool configurations such as rustfmt.toml?
-> Also see Clippy.toml

TODO: Try to think of a more general approach


## Future possibilities

* It might be desirable to also be able to configure the settings of specifics lints. Such as via the `clippy.toml`.
* It would make sense to include other settings related to lints, such as output verbosity or `--cap-lints` in the lint configuration.

## TODO text

Things that still need work or aren't even included in the text, yet:

1. Difference between 'workspace level' and 'individual packages'?
1. Check if .toml syntax in examples is correct
1. Find out what `.cargo/config` is currently used for. Write down pros/cons of putting the lint config into `.cargo/config`. Add this to the appropriate section.
1. In general expand the Pros/Cons of the configuration file section
1. What about lint groups? Rustc now also has the `cargo` lint group as well as the
   `rust_2018_ideoms` and `rust_2018_compatibilty` lint groups.
1. How does precedence work? Can packages override workspaces, or the other way around? Or maybe based on strictness (workspaces can make lints more restrictive, not less)?
1. How much of an issue is errors for unknown lints? My feeling is that it
   should be OK, but it does set a floor for the minimum supported rustc for
   something that is not really critical.
1. How important is it to report the location of the lint? Currently rustc will
   tell you it's in the command-line. Finding the correct location for the lint
   may be tricky in a large project. I personally think it would be good to have
   the intent to implement location reporting in rustc before stabilization.
   TODO @oli_obk made a suggestion for this somewhere on GitHub, need to find again

## Unresolved Questions

TODO

## Before publishing on IRLO

1. Ensure all github code links are permalinks
1. Ask in #wg-clippy if anyone want's to cross-check it
1. Run vim spellchecker, codespell and paste into Google Docs to avoid typos

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
