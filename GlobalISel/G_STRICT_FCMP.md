# G_STRICT_FCMP

### GenericOpcodes.td

```cpp
def G_STRICT_FLDEXP : ConstrainedInstruction<G_FLDEXP>;
def G_STRICT_FCMP : ConstrainedInstruction<G_FCMP>;
def G_STRICT_FCMPS : ConstrainedInstruction<G_FCMP>;
```

### TargetOpcodes.def

- Used in `TargetOpcodes.h` to create gMIR OpCode
- `TargetOpcodes.h` is about MIR
- Avoiding RTTI

```cpp
HANDLE_TARGET_OPCODE(G_STRICT_FLDEXP)
HANDLE_TARGET_OPCODE(G_STRICT_FCMP)
HANDLE_TARGET_OPCODE(G_STRICT_FCMPS)
```

### IRTranslator.cpp

- Case for `Intrinsics.h` and `IntrinsicEnums.inc`

```cpp
static unsigned getConstrainedOpcode(Intrinsic::ID ID) {
  switch (ID) {
  case Intrinsic::experimental_constrained_fadd:
    return TargetOpcode::G_STRICT_FADD;
  case Intrinsic::experimental_constrained_fsub:
    return TargetOpcode::G_STRICT_FSUB;
  case Intrinsic::experimental_constrained_fmul:
    return TargetOpcode::G_STRICT_FMUL;
  case Intrinsic::experimental_constrained_fdiv:
    return TargetOpcode::G_STRICT_FDIV;
  case Intrinsic::experimental_constrained_frem:
    return TargetOpcode::G_STRICT_FREM;
  case Intrinsic::experimental_constrained_fma:
    return TargetOpcode::G_STRICT_FMA;
  case Intrinsic::experimental_constrained_sqrt:
    return TargetOpcode::G_STRICT_FSQRT;
  case Intrinsic::experimental_constrained_ldexp:
    return TargetOpcode::G_STRICT_FLDEXP;
  case Intrinsic::experimental_constrained_fcmp:
    return TargetOpcode::G_STRICT_FCMP;
  case Intrinsic::experimental_constrained_fcmps:
    return TargetOpcode::G_STRICT_FCMPS;
  default:
    return 0;
  }
}
```

```cpp
bool IRTranslator::translateConstrainedFPIntrinsic(
    const ConstrainedFPIntrinsic &FPI, MachineIRBuilder &MIRBuilder) {
  fp::ExceptionBehavior EB = *FPI.getExceptionBehavior();

  unsigned Opcode = getConstrainedOpcode(FPI.getIntrinsicID());
  if (!Opcode)
    return false;

  uint32_t Flags = MachineInstr::copyFlagsFromInstruction(FPI);
  if (EB == fp::ExceptionBehavior::ebIgnore)
    Flags |= MachineInstr::NoFPExcept;


  if (Opcode == TargetOpcode::G_STRICT_FCMP ||
      Opcode == TargetOpcode::G_STRICT_FCMPS) {
    auto *FPCmp = cast<ConstrainedFPCmpIntrinsic>(&FPI);
    Register Operand0 = getOrCreateVReg(*FPCmp->getArgOperand(0));
    Register Operand1 = getOrCreateVReg(*FPCmp->getArgOperand(1));
    Register Result = getOrCreateVReg(FPI);
    MIRBuilder.buildInstr(Opcode, {Result}, {}, Flags)
        .addPredicate(FPCmp->getPredicate())
        .addUse(Operand0)
        .addUse(Operand1);
    return true;
  }

  SmallVector<llvm::SrcOp, 4> VRegs;
  for (unsigned I = 0, E = FPI.getNonMetadataArgCount(); I != E; ++I)
    VRegs.push_back(getOrCreateVReg(*FPI.getArgOperand(I)));

  MIRBuilder.buildInstr(Opcode, {getOrCreateVReg(FPI)}, VRegs, Flags);
  return true;
}
```

### irtranslator-constrained-fcmp.ll

- PC
```bash
~/Desktop/llvm-project/build/bin/llvm-lit <test.ll>
```

- Cloud

```bash
~/llvm-project/build/bin/llvm-lit <test.ll>
```

```ll
; RUN: llc --global-isel --mtriple=aarch64-- --stop-after=irtranslator -o - %s | FileCheck %s

define void @test_strict_fcmp(float %a, float %b) strictfp {
  %r0 = call i1 @llvm.experimental.constrained.fcmp.f32(float %a, float %b, metadata !"oeq", metadata !"fpexcept.strict")
  %r1 = call i1 @llvm.experimental.constrained.fcmp.f32(float %a, float %b, metadata !"ogt", metadata !"fpexcept.strict")
  %r2 = call i1 @llvm.experimental.constrained.fcmp.f32(float %a, float %b, metadata !"oge", metadata !"fpexcept.strict")
  %r3 = call i1 @llvm.experimental.constrained.fcmp.f32(float %a, float %b, metadata !"olt", metadata !"fpexcept.strict")
  %r4 = call i1 @llvm.experimental.constrained.fcmp.f32(float %a, float %b, metadata !"ole", metadata !"fpexcept.strict")
  %r5 = call i1 @llvm.experimental.constrained.fcmp.f32(float %a, float %b, metadata !"one", metadata !"fpexcept.strict")
  %r6 = call i1 @llvm.experimental.constrained.fcmp.f32(float %a, float %b, metadata !"ord", metadata !"fpexcept.strict")
  %r7 = call i1 @llvm.experimental.constrained.fcmp.f32(float %a, float %b, metadata !"uno", metadata !"fpexcept.strict")
  %r8 = call i1 @llvm.experimental.constrained.fcmp.f32(float %a, float %b, metadata !"ueq", metadata !"fpexcept.strict")
  %r9 = call i1 @llvm.experimental.constrained.fcmp.f32(float %a, float %b, metadata !"ugt", metadata !"fpexcept.strict")
  %r10 = call i1 @llvm.experimental.constrained.fcmp.f32(float %a, float %b, metadata !"uge", metadata !"fpexcept.strict")
  %r11 = call i1 @llvm.experimental.constrained.fcmp.f32(float %a, float %b, metadata !"ult", metadata !"fpexcept.strict")
  %r12 = call i1 @llvm.experimental.constrained.fcmp.f32(float %a, float %b, metadata !"ule", metadata !"fpexcept.strict")
  %r13 = call i1 @llvm.experimental.constrained.fcmp.f32(float %a, float %b, metadata !"une", metadata !"fpexcept.strict")
  ret void
; CHECK: G_STRICT_FCMP floatpred(oeq)
; CHECK: G_STRICT_FCMP floatpred(ogt)
; CHECK: G_STRICT_FCMP floatpred(oge)
; CHECK: G_STRICT_FCMP floatpred(olt)
; CHECK: G_STRICT_FCMP floatpred(ole)
; CHECK: G_STRICT_FCMP floatpred(one)
; CHECK: G_STRICT_FCMP floatpred(ord)
; CHECK: G_STRICT_FCMP floatpred(uno)
; CHECK: G_STRICT_FCMP floatpred(ueq)
; CHECK: G_STRICT_FCMP floatpred(ugt)
; CHECK: G_STRICT_FCMP floatpred(uge)
; CHECK: G_STRICT_FCMP floatpred(ult)
; CHECK: G_STRICT_FCMP floatpred(ule)
; CHECK: G_STRICT_FCMP floatpred(une)
}

define void @test_strict_fcmps(float %a, float %b) strictfp {
  %r0 = call i1 @llvm.experimental.constrained.fcmps.f32(float %a, float %b, metadata !"oeq", metadata !"fpexcept.strict")
  %r1 = call i1 @llvm.experimental.constrained.fcmps.f32(float %a, float %b, metadata !"ogt", metadata !"fpexcept.strict")
  %r2 = call i1 @llvm.experimental.constrained.fcmps.f32(float %a, float %b, metadata !"oge", metadata !"fpexcept.strict")
  %r3 = call i1 @llvm.experimental.constrained.fcmps.f32(float %a, float %b, metadata !"olt", metadata !"fpexcept.strict")
  %r4 = call i1 @llvm.experimental.constrained.fcmps.f32(float %a, float %b, metadata !"ole", metadata !"fpexcept.strict")
  %r5 = call i1 @llvm.experimental.constrained.fcmps.f32(float %a, float %b, metadata !"one", metadata !"fpexcept.strict")
  %r6 = call i1 @llvm.experimental.constrained.fcmps.f32(float %a, float %b, metadata !"ord", metadata !"fpexcept.strict")
  %r7 = call i1 @llvm.experimental.constrained.fcmps.f32(float %a, float %b, metadata !"uno", metadata !"fpexcept.strict")
  %r8 = call i1 @llvm.experimental.constrained.fcmps.f32(float %a, float %b, metadata !"ueq", metadata !"fpexcept.strict")
  %r9 = call i1 @llvm.experimental.constrained.fcmps.f32(float %a, float %b, metadata !"ugt", metadata !"fpexcept.strict")
  %r10 = call i1 @llvm.experimental.constrained.fcmps.f32(float %a, float %b, metadata !"uge", metadata !"fpexcept.strict")
  %r11 = call i1 @llvm.experimental.constrained.fcmps.f32(float %a, float %b, metadata !"ult", metadata !"fpexcept.strict")
  %r12 = call i1 @llvm.experimental.constrained.fcmps.f32(float %a, float %b, metadata !"ule", metadata !"fpexcept.strict")
  %r13 = call i1 @llvm.experimental.constrained.fcmps.f32(float %a, float %b, metadata !"une", metadata !"fpexcept.strict")
  ret void
; CHECK: G_STRICT_FCMPS floatpred(oeq)
; CHECK: G_STRICT_FCMPS floatpred(ogt)
; CHECK: G_STRICT_FCMPS floatpred(oge)
; CHECK: G_STRICT_FCMPS floatpred(olt)
; CHECK: G_STRICT_FCMPS floatpred(ole)
; CHECK: G_STRICT_FCMPS floatpred(one)
; CHECK: G_STRICT_FCMPS floatpred(ord)
; CHECK: G_STRICT_FCMPS floatpred(uno)
; CHECK: G_STRICT_FCMPS floatpred(ueq)
; CHECK: G_STRICT_FCMPS floatpred(ugt)
; CHECK: G_STRICT_FCMPS floatpred(uge)
; CHECK: G_STRICT_FCMPS floatpred(ult)
; CHECK: G_STRICT_FCMPS floatpred(ule)
; CHECK: G_STRICT_FCMPS floatpred(une)
}

```

### legalizer-info-validation.mir

```bash
# DEBUG-NEXT: G_STRICT_FLDEXP (opcode {{[0-9]+}}): 2 type indices, 0 imm indices
# DEBUG-NEXT:.. type index coverage check SKIPPED: no rules defined
# DEBUG-NEXT:.. imm index coverage check SKIPPED: no rules defined
# DEBUG-NEXT: G_STRICT_FCMP (opcode {{[0-9]+}}): 2 type indices, 0 imm indices
# DEBUG-NEXT: .. type index coverage check SKIPPED: no rules defined
# DEBUG-NEXT: .. imm index coverage check SKIPPED: no rules defined
# DEBUG-NEXT: G_STRICT_FCMPS (opcode {{[0-9]+}}): 2 type indices, 0 imm indices
# DEBUG-NEXT: .. type index coverage check SKIPPED: no rules defined
# DEBUG-NEXT: .. imm index coverage check SKIPPED: no rules defined
```
