# __builtin_fpclassify

### lib/CodeGen/CGBuiltin.cpp

```cpp
  case Builtin::BI__builtin_fpclassify: {
    CodeGenFunction::CGFPOptionsRAII FPOptsRAII(*this, E);

    Value *NanLiteral = EmitScalarExpr(E->getArg(0));
    Value *InfiniteLiteral = EmitScalarExpr(E->getArg(1));
    Value *NormalLiteral = EmitScalarExpr(E->getArg(2));
    Value *SubnormalLiteral = EmitScalarExpr(E->getArg(3));
    Value *ZeroLiteral = EmitScalarExpr(E->getArg(4));
    Value *V = EmitScalarExpr(E->getArg(5));

    BasicBlock *Entry = Builder.GetInsertBlock();
    BasicBlock *NotNan = createBasicBlock("fpclassify_not_nan", CurFn);
    BasicBlock *NotInfinite =
        createBasicBlock("fpclassify_not_infinite", CurFn);
    BasicBlock *NotNormal = createBasicBlock("fpclassify_not_normal", CurFn);
    BasicBlock *NotSubnormal =
        createBasicBlock("fpclassify_not_subnormal", CurFn);
    BasicBlock *End = createBasicBlock("fpclassify_end", CurFn);

    // End Block
    Builder.SetInsertPoint(End);
    PHINode *Result = Builder.CreatePHI(ConvertType(E->getArg(0)->getType()), 5,
                                        "fpclassify_result");

    // Entry Block
    Builder.SetInsertPoint(Entry);
    Value *IsNan = Builder.createIsFPClass(V, FPClassTest::fcNan);
    Result->addIncoming(NanLiteral, Entry);
    Builder.CreateCondBr(IsNan, End, NotNan);

    // NotNan Block
    Builder.SetInsertPoint(NotNan);
    Value *IsInfinite = Builder.createIsFPClass(V, FPClassTest::fcInf);
    Result->addIncoming(InfiniteLiteral, NotNan);
    Builder.CreateCondBr(IsInfinite, End, NotInfinite);

    // NotInfinite Block
    Builder.SetInsertPoint(NotInfinite);
    Value *IsNormal = Builder.createIsFPClass(V, FPClassTest::fcNormal);
    Result->addIncoming(NormalLiteral, NotInfinite);
    Builder.CreateCondBr(IsNormal, End, NotNormal);

    // NotNormal Block
    Builder.SetInsertPoint(NotNormal);
    Value *IsSubnormal = Builder.createIsFPClass(V, FPClassTest::fcSubnormal);
    Result->addIncoming(SubnormalLiteral, NotNormal);
    Builder.CreateCondBr(IsSubnormal, End, NotSubnormal);

    // NotSubnormal Block
    Builder.SetInsertPoint(NotSubnormal);
    Result->addIncoming(ZeroLiteral, NotSubnormal);
    Builder.CreateBr(End);

    // End Block
    Builder.SetInsertPoint(End);
    return RValue::get(Result);
  }
```
