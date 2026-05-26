# Optimize shl lowering by removing G_ZEXT

## llvm/lib/Target/AArch64/AArch64Combine.td

```cpp
// before:
//   %b32:_(s32) = G_ZEXT %b:_(s8)
//   %r:_(s32)   = G_SHL  %a:_(s32), %b32:_(s32)
// after:
//   %r:_(s32)   = G_SHL  %a:_(s32), (G_ANYEXT %b)
def remove_zext_when_lowering_shl : GICombineRule<
  (defs root:$root, register_matchinfo:$matchinfo),
  (match (wip_match_opcode G_SHL, G_LSHR, G_ASHR):$root,
         [{ return matchRemoveZextWhenLoweringShift(*${root}, MRI, ${matchinfo}); }]),
  (apply [{ applyRemoveZextWhenLoweringShift(*${root}, MRI, B, ${matchinfo}); }])
>;

// Post-legalization combines which are primarily optimizations.
def AArch64PostLegalizerCombiner
    : GICombiner<"AArch64PostLegalizerCombinerImpl",
                       [copy_prop, cast_of_cast_combines, constant_fold_fp_ops,
                        ...
                        remove_zext_when_lowering_shl,
```

## 

### Reference
- LSL instruction: https://developer.arm.com/documentation/ddi0602/2026-03/Base-Instructions/LSL--register---Logical-shift-left--register---an-alias-of-LSLV-?lang=en
