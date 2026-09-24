# test-regression-runner

> Run `./run_regression_tests.sh --full --no-boot --filter <word>` for targeted checks; `--filter` narrows the selected mode, it does not select one.

## Why It Matters

`regression/` and `stress/` run only in `--full` mode, so `--filter structs` alone (quick mode) selects **nothing** and passes vacuously.

## Commands

```bash
./run_regression_tests.sh --quick                        # default: unit test, smoke, LSP protocol
./run_regression_tests.sh --full                         # ./boot.sh first, then every category
./run_regression_tests.sh --full --no-boot               # every category, skip fixed point
./run_regression_tests.sh --full --no-boot --filter structs
./run_regression_tests.sh --full --no-boot --filter balanced-parens   # after lexer/parser edits
./run_regression_tests.sh --full --filter interpreter    # REPL interpreter vs compiled output
./run_regression_tests.sh --full --no-boot --filter math # crypto vectors
./run_regression_tests.sh --dry-run --full --filter x    # list exactly what would run
```

Other flags: `--boot`, `--verbose`, `--timeout N`.

## Categories

smoke, regression (the `test` harness), interpreter (differential: ICNF interpreter vs binary), stress, integration, compile-fail, packages, packages-fail, packages-build, scripts, lsp, unit test.

## Notes

- Before touching struct-related compiler code (`ast`, `codegen`, `icnf`, `type_annotate`, `parser`, `region_inference`) run `--full --no-boot --filter structs`.

## See Also

- [boot-fixed-point-workflow](boot-fixed-point-workflow.md)
