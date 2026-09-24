# tool-cli-arguments

> Put the source file first: `zyl file.zyl [-o out] [--emit-asm]`; everything else is a subcommand, and unknown words become the output path.

## Why It Matters

The argument parser is strict about order and loose about content:

- `zyl --emit-asm prog.zyl` takes the flag as the source path → "cannot open source file".
- `zyl prog.zyl --emit-icnf` builds a binary **named** `--emit-icnf`.
- `zyl --help` is treated as a file name; use `zyl help`.
- The raw compiler (`stage2.bin`, `build/boot/zyl-self`) with **no arguments** runs a legacy boot protocol (compiles `/tmp/zyl_boot_in.zyl`); the installed `zyl` wrapper with no arguments starts the REPL.

## Commands

```bash
zyl hello.zyl                          # writes hello.s, links ./hello
zyl hello.zyl -o bin/hello             # bin/hello.s + bin/hello
zyl hello.zyl -o hello.s --emit-asm    # assembly only, exactly at -o
zyl hello.zyl --error-format=json      # diagnostics as JSON lines on stderr
zyl hello.zyl --contracts=warn         # contract profile: strict (default)|debug|warn|off|production
zyl eval hello.zyl                     # run via ICNF interpreter, no binary
zyl repl
zyl doc [file.zyl | dir] [-o out.md]   # Markdown API docs from comments (stdout without -o)
zyl new owner/name | add NAME [VER] | fetch | build [--locked] | test
zyl update | vendor | audit | key | help
zyl publish [--index DIR [--url-base URL]]
```

## `zyl doc`

- The file's leading comment block is the module doc: its `; === Title ===` line is the title, `Module:` lines are dropped.
- The contiguous `;` lines directly above a top-level `defn`/`def`/`deftype`/`defstruct`/`trait`/`defmacro` are its doc; a blank line or a `; ===`/`; ---` separator ends the block. If the block has any `;|` lines, only those are used (so ordinary comments can sit beside a doc).
- Each definition is listed in source order with its signature. In a package (a `zyl.pkg` beside the file or in the directory) only `pub` definitions appear.
- A directory is walked recursively, files sorted (deterministic output).

## Notes

- A first argument ending in `.zyl` or starting with `-` means "compile this file".
- The compiler chdirs to its bundle directory before compiling; relative paths resolve against where you ran it.
- Package subcommands search upward for `zyl.pkg` (`E_MANIFEST_NOT_FOUND`).
- The only compile flags are `-o`, `--emit-asm`, `--error-format=json` and `--contracts=P` (accepted anywhere on the line): no `--emit-ast/-icnf/-expanded/-typed`.
- `zyl build`/`zyl test` reuse a cached build (`~/.zyl/cache/<key>`) when no input changed; `ZYL_NO_BUILD_CACHE=1` forces a compile.
- Environment: `ZYL_HOME` (bundle dir), `ZYL_DEBUG_STAGES` (append stage names to `/tmp/dbg`), `ZYL_MAX_MEMORY` (budget bytes; 0 disables), `ZYL_INDEX` (package index: git URL or local path), `ZYL_NO_BUILD_CACHE=1`, `ZYL_STAGE_TIMEOUT` (boot stages, default 2400 s), `ZYL_STAGE_MEMORY` (boot stages' allocation ceiling, default 2 GB).

## See Also

- [tool-debugging-the-pipeline](tool-debugging-the-pipeline.md)
- [pkg-stdlib-resolution](pkg-stdlib-resolution.md)
