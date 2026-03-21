# builtin_fpclassify

```cpp
  case Builtin::BI__builtin_fpclassify: {
    CIRGenFunction::CIRGenFPOptionsRAII FPOptsRAII(*this, e);
    mlir::Location loc = getLoc(e->getBeginLoc());

    mlir::Value value = emitScalarExpr(e->getArg(5));

    mlir::Type resultTy = convertType(e->getType());

    auto is_zero =
        cir::IsFPClassOp::create(builder, loc, value, cir::FPClassTest::fcZero);
    cir::TernaryOp::create(
        builder, loc, is_zero,
        [&](mlir::OpBuilder &b, mlir::Location l) {
          mlir::Value ZeroLiteral = emitScalarExpr(e->getArg(4));
          cir::ConstantOp fp_zero = cir::ConstantOp::create(b, l, ZeroLiteral);
          cir::YieldOp::create(b, l, fp_zero);
        },
        [&](mlir::OpBuilder &b, mlir::Location l) {
          auto is_nan =
              cir::IsFPClassOp::create(b, l, value, cir::FPClassTest::fcNan);
          cir::TernaryOp::create(
              b, l, is_nan,
              [&](mlir::OpBuilder &b, mlir::Location l) {
                mlir::Value NanLiteral = emitScalarExpr(e->getArg(0));
                cir::ConstantOp fp_nan =
                    cir::ConstantOp::create(b, l, NanLiteral);
                cir::YieldOp::create(b, l, fp_nan);
              },
              [&](mlir::OpBuilder &b, mlir::Location l) {
                auto is_infinity = cir::ConstantOp::create(
                    b, l, value, cir::FPClassTest::fcInf);
                cir::TernaryOp::create(
                    b, l, is_infinity,
                    [&](mlir::OpBuilder &b, mlir::Location l) {
                      mlir::Value InfinityLiteral =
                          emitScalarExpr(e->getArg(1));
                      cir::ConstantOp fp_infinite =
                          cir::ConstantOp::create(b, l, InfinityLiteral);
                      cir::YieldOp::create(b, l, fp_infinite);
                    },
                    [&](mlir::OpBuilder &b, mlir::Location l) {
                      auto is_normal = cir::IsFPClassOp::create(
                          b, l, value, cir::FPClassTest::fcNormal);

                      mlir::Value NormalLiteral = emitScalarExpr(e->getArg(2));
                      cir::ConstantOp fp_normal =
                          cir::ConstantOp::create(b, l, NormalLiteral);

                      mlir::Value SubnormalLiteral =
                          emitScalarExpr(e->getArg(3));
                      cir::ConstantOp fp_subnormal =
                          cir::ContantOp::create(b, l, SubnormalLiteral);

                      mlir::Value returnValue = cir::SelectOp::create(
                          b, l, resultTy, is_normal, fp_normal, fp_subnormal);

                      cir::YieldOp::create(b, l, returnValue);
                    });
              });
        });
  }
```
