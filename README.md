# CIR practice

## Command
- Converting C/C++ to CIR

```bash
/tmp/install-llvm/bin/clang -fclangir -emit-cir -clangir-disable-passes  main.c -o main.cir
```

- Converting CIR to LLVM IR

```bash
/tmp/install-llvm/bin/cir-translate --cir-to-llvmir main.cir -o main.ll
```

- Converting C/C++ to LLVM IR

```bash
/tmp/install-llvm/bin/clang -fclangir -S -emit-llvm main.c
```

```cpp
CodeGenFunction::CGFPOptionsRAII FPOptsRAII(*this, E);
Value *V = EmitScalarExpr(E->getArg(5));
llvm::Type *Ty = ConvertType(E->getArg(5)->getType());

Value *NanLiteral = EmitScalarExpr(E->getArg(0));
Value *InfLiteral = EmitScalarExpr(E->getArg(1));
Value *NormalLiteral = EmitScalarExpr(E->getArg(2));
Value *SubnormalLiteral = EmitScalarExpr(E->getArg(3));
Value *ZeroLiteral = EmitScalarExpr(E->getArg(4));

Value *IsNan       = Builder.createIsFPClass(V, 0b0000000011);
Value *IsInf       = Builder.createIsFPClass(V, 0b1000000100);
Value *IsNormal    = Builder.createIsFPClass(V, 0b0100001000);
Value *IsSubnormal = Builder.createIsFPClass(V, 0b0010010000);
Value *IsZero      = Builder.createIsFPClass(V, 0b0001100000);

BasicBlock *Begin = Builder.GetInsertBlock();

// Create PHI Node
BasicBlock *End = createBasicBlock("fpclassify_end",this->CurFn);
Builder.SetInsertPoint(End);
PHINode *Result = Builder.CreatePHI(ConvertType(E->getArg(0)->getType()), 5,"fpclassify_result");

// Check IsNan
Builder.SetInsertPoint(Begin);
BasicBlock *NotNan = createBasicBlock("fpclassify_not_nan", this->CurFn);
Builder.CreateCondBr(IsNan, End, NotNan);
Result->addIncoming(NanLiteral,Begin);

// Check IsInfinity
Builder.SetInsertPoint(NotNan);
BasicBlock *NotInf = createBasicBlock("fpclassify_not_inf", this->CurFn);
Builder.CreateCondBr(IsInf, End, NotInf);
Result->addIncoming(InfLiteral, NotNan);

// Check IsNormal
Builder.SetInsertPoint(NotInf);
BasicBlock *NotNormal = createBasicBlock("fpclassify_not_normal",this->CurFn);
Builder.CreateCondBr(IsNormal, End, NotNormal);
Result->addIncoming(NormalLiteral, NotInf);

// Check IsSubnormal
Builder.SetInsertPoint(NotNormal);
BasicBlock *NotSubnormal = createBasicBlock("fpclassify_not_subnormal", this->CurFn);
Builder.CreateCondBr(IsSubnormal, End, NotSubnormal);
Result->addIncoming(SubnormalLiteral, NotNormal);

// Check IsZero
Builder.SetInsertPoint(NotSubnormal);
Builder.CreateBr(End);
Result->addIncoming(ZeroLiteral, NotSubnormal);

Builder.SetInsertPoint(End);
return RValue::get(Result);
```
