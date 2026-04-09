# GlobalISel Implementation Notes

## Running Test

```bash
ninja check-llvm-codegen-aarch64-globalisel
```

```bash
ninja check-llvm-codegen-riscv-globalisel
```

## Update FileCheck Test File

```bash
python3 update_mir_test_checks.py --llc-binary build/bin/llc <test file>
```
