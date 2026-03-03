# signbit function


```cpp
  case Builtin::BI__builtin_signbitl: {
    CIRGenFunction::CIRGenFPOptionsRAII fpOptsRaii(*this, e);
    mlir::Value v = emitScalarExpr(e->getArg(0));
    mlir::Location loc = getLoc(e->getBeginLoc());
    auto signBitOp = cir::SignBitOp::create(builder, loc, v);
    return RValue::get(
        builder.createBoolToInt(signBitOp, convertType(e->getType())));
  }
```
