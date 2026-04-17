# Make gMIR->MIR Instruction Selection to Process newly created block while Selection

## algorithm

```python
BlockSuccessorList ← [Input Block]

while BlockSuccessorList is not empty:
    CurrentBlock ← last block from BlockSuccessorList
    CurrentBlockSuccessorIterator ← CurrentBlock's successor iterator

    if CurrentBlockSuccessorIterator has remaining successors:
        Add the successor block to BlockSuccessorList
        continue

    VirtualRegisterCache ← current state of virtual registers

    if a new block is created while instruction selection:
        revert virtual register state using VirtualRegisterCache
        put CurrentBlock back to BlockSuccessorList
    else
        Remove the rightmost block from BlockSuccessorList

```

## lib/CodeGen/GlobalISel/InstructionSelect.cpp

```cpp
#include "llvm/CodeGen/GlobalISel/InstructionSelect.h"
#include "llvm/ADT/DenseMap.h"
#include "llvm/ADT/DenseSet.h"
...

    GISelObserverWrapper AllObservers;
    MIIteratorMaintainer MIIMaintainer;
    AllObservers.addObserver(&MIIMaintainer);
    RAIIDelegateInstaller DelInstaller(MF, &AllObservers);
    ISel->AllObservers = &AllObservers;

    SmallVector<
        std::pair<MachineBasicBlock *, MachineBasicBlock::succ_iterator>>
        BlockSuccessorList;
    DenseSet<MachineBasicBlock *> VisitedBlocks;

    auto AppendBlockSuccessor = [&](MachineBasicBlock *Block) {
      if (!VisitedBlocks.contains(Block)) {
        VisitedBlocks.insert(Block);
        BlockSuccessorList.emplace_back(Block, Block->succ_begin());
      }
    };
    AppendBlockSuccessor(&MF.front());

    while (!BlockSuccessorList.empty()) {
      MachineBasicBlock *&CurrentBlock = BlockSuccessorList.back().first;
      MachineBasicBlock::succ_iterator &CurrentBlockSuccessorIterator =
          BlockSuccessorList.back().second;

      if (CurrentBlockSuccessorIterator != CurrentBlock->succ_end()) {
        MachineBasicBlock *CurrentBlockSuccessor =
            *CurrentBlockSuccessorIterator++;
        AppendBlockSuccessor(CurrentBlockSuccessor);
        continue;
      }

      DenseMap<Register, RegClassOrRegBank> VirtualRegisterCache;
      for (const MachineInstr &Instruction : *CurrentBlock)
        for (const MachineOperand &Operand : Instruction.operands())
          if (Operand.isReg() && Operand.getReg().isVirtual())
            VirtualRegisterCache.try_emplace(
                Operand.getReg(), MRI.getRegClassOrRegBank(Operand.getReg()));

      const size_t BlockCountBeforeSelection = MF.size();

      ISel->CurMBB = CurrentBlock;
      SelectedBlocks.insert(CurrentBlock);

      MIIMaintainer.MII = CurrentBlock->rbegin();
      for (auto End = CurrentBlock->rend(); MIIMaintainer.MII != End;) {
        MachineInstr &MI = *MIIMaintainer.MII;
        ++MIIMaintainer.MII;

        LLVM_DEBUG(dbgs() << "\nSelect:  " << MI);
        if (!selectInstr(MI)) {
          LLVM_DEBUG(dbgs() << "Selection failed!\n";
                     MIIMaintainer.reportFullyCreatedInstrs());
          reportGISelFailure(MF, MORE, "gisel-select", "cannot select", MI);
          return false;
        }
        LLVM_DEBUG(MIIMaintainer.reportFullyCreatedInstrs());
      }

      if (MF.size() > BlockCountBeforeSelection) {
        for (auto &VirtualRegisterEntry : VirtualRegisterCache)
          MRI.setRegClassOrRegBank(VirtualRegisterEntry.first,
                                   VirtualRegisterEntry.second);
        SelectedBlocks.erase(CurrentBlock);
        BlockSuccessorList.pop_back();
        VisitedBlocks.erase(CurrentBlock);
        AppendBlockSuccessor(CurrentBlock);
        continue;
      }
      BlockSuccessorList.pop_back();
    }
  }

  for (MachineBasicBlock &MBB : MF) {
    if (MBB.empty())
      continue;
```

## unittests/CodeGen/GlobalISel/InstructionSelectTest.cpp

```cpp
#include "llvm/CodeGen/GlobalISel/InstructionSelect.h"
#include "GISelMITest.h"
#include "llvm/CodeGen/GlobalISel/InstructionSelector.h"
namespace {
struct CustomISel : public InstructionSelector {
  bool select(MachineInstr &MI) override {
    static bool Triggered = false;
    if (!Triggered) {
      Triggered = true;
      auto &MF = *MI.getMF();
      MF.push_back(MF.CreateMachineBasicBlock());
      MI.getParent()->addSuccessor(&MF.back());
    }
    return true;
  }
  void setupGeneratedPerFunctionState(MachineFunction &) override {}
};
TEST_F(AArch64GISelMITest, NewBlockWhileInstructionSelection) {
  setUp(R"(
   $x0 = COPY %2(s64)
)");
  if (!TM)
    GTEST_SKIP();
  CustomISel ISel;
  InstructionSelect Pass;
  Pass.setInstructionSelector(&ISel);
  ASSERT_EQ(MF->size(), 1u);
  Pass.selectMachineFunction(*MF);
  EXPECT_EQ(MF->size(), 2u);
}
} // namespace

```

## unittests/CodeGen/GlobalISel/CMakeLists.txt

```cpp
add_llvm_unittest(GlobalISelTests
  IRTranslatorBF16Test.cpp
  ConstantFoldingTest.cpp
  CSETest.cpp
  GIMatchTableExecutorTest.cpp
  LegalizerTest.cpp
  LegalizerHelperTest.cpp
  LegalizerInfoTest.cpp
  MachineIRBuilderTest.cpp
  GISelMITest.cpp
  PatternMatchTest.cpp
  KnownBitsTest.cpp
  KnownFPClassTest.cpp
  KnownBitsVectorTest.cpp
  GISelUtilsTest.cpp
  GISelAliasTest.cpp
  CallLowering.cpp
  InstructionSelectTest.cpp
  InstructionSelectionTest.cpp
  )
```
