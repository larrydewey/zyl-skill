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
zyl eval hello.zyl                     # run via ICNF interpreter, no binary
zyl repl
zyl new owner/name | add NAME [VER] | fetch | build [--locked] | test
zyl update | vendor | audit | publish | key | help
```

## Notes

- A first argument ending in `.zyl` or starting with `-` means "compile this file".
- The compiler chdirs to its bundle directory before compiling; relative paths resolve against where you ran it.
- Package subcommands search upward for `zyl.pkg` (`E_MANIFEST_NOT_FOUND`).
- Only `--emit-asm` exists: no `--emit-ast/-icnf/-expanded/-typed`.
- Environment: `ZYL_HOME` (bundle dir), `ZYL_DEBUG_STAGES` (append stage names to `/tmp/dbg`), `ZYL_MAX_MEMORY` (budget bytes; 0 disables), `ZYL_STAGE_TIMEOUT` (boot stages, default 2400 s).

## See Also

- [tool-debugging-the-pipeline](tool-debugging-the-pipeline.md)
- [pkg-stdlib-resolution](pkg-stdlib-resolution.md)
