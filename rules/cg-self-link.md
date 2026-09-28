# cg-self-link

> Emit only instruction and operand forms `asm_x86.zyl` encodes: a freestanding program is assembled and linked by the compiler itself (`asm_x86.zyl`, `elf_link.zyl`) against `rt.zo`, so a new form in codegen or in a runtime primitive's expansion must be taught to the assembler first and checked against GNU as.

## Why It Matters

Every program with no foreign `ffi-call` and no native objects links with no `cc`, `as` or `ld`. The assembler parses exactly the Intel-noprefix text the backends emit and encodes it; per instruction, its bytes match GNU as. A form it does not know stops the link with `E_ASM_UNSUPPORTED`, and a form it knows but encodes wrongly is a silent miscompile that only freestanding binaries show (the compiler stages themselves link with `cc`). Before the encoder fixes of 2026-09-28, `spl`/`bpl`/`sil`/`dil` in the reg field without REX read `ah`..`bh`, and `1add8ba` had to route stores through `cl` and avoid `cmovge` until the encoder learned them.

## Checklist for a new emitted form

1. Add the encoding to `asm_x86.zyl` (REX/VEX prefixes, ModRM/SIB, `index*scale` displacements, relocation kind).
2. Run `tests/scripts/asm-oracle.sh` (an every-register x every-form sweep diffed against GNU as, bytes and relocation sites) and add the case to `tests/regression/asm-x86.zyl`.
3. Only then emit the form from `codegen.zyl` or use it in a `%` primitive's expansion.
4. Reseed; the runtime's `rt.s` changes the key of `rt.zo`, which is rebuilt automatically.

## Notes

- `ZYL_EXTERNAL_LD=1` links with `cc -nostdlib -static` instead: the quickest way to tell an assembler bug from a codegen bug.
- Jumps are always encoded rel32; there is no relaxation pass.
- `rt.zo` is keyed by the BLAKE3 of `rt.s` + `start.s` and carries its length; a stale or torn cache is rebuilt in memory, and a hit and a miss give byte-identical binaries.
- A strong symbol neither side defines is `E_LINK_UNDEFINED` (a GOT symbol: `E_LINK_UNDEFINED_GOT`); weak undefined symbols resolve to 0.

## See Also

- [cg-native-backend-mir](cg-native-backend-mir.md)
- [cg-symbols-and-entry](cg-symbols-and-entry.md)
- [boot-runtime-module](boot-runtime-module.md)
