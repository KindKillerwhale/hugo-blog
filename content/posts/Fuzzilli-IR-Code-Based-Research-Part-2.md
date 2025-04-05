+++
title = "Fuzzilli IR Code Based Research Part 2 ( Opcodes.swift, Operation.swift, Program.swift, Variable.swift )"
description = """A deep dive into the core IR components of Fuzzilli, focusing on Opcodes.swift, Operation.swift, Program.swift, Variable.swift. This post is the second in the series exploring the IR internal structure of Fuzzilli.
"""
draft = false
date = "2025-04-05"
author = "DongHyeon Hwang ( @kind_killerwhale )"
tags = ['Fuzzilli', 'IR', 'Fuzzer']
+++

&nbsp;

Welcome back to the second post in this series. In this article, we’ll dive into `Opcodes.swift`, `Operation.swift`, `Program.swift`, and `Variable.swift`.

# Opcodes.swift
---------------

```swift
// Copyright 2023 Google LLC
//
// Licensed under the Apache License, Version 2.0 (the "License");
// you may not use this file except in compliance with the License.
// You may obtain a copy of the License at
//
// https://www.apache.org/licenses/LICENSE-2.0
//
// Unless required by applicable law or agreed to in writing, software
// distributed under the License is distributed on an "AS IS" BASIS,
// WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
// See the License for the specific language governing permissions and
// limitations under the License.

/// Enum defining all opcodes supported in FuzzIL.
///
/// There should be a 1:1 mapping between opcodes and Operation subclasses.
/// This enum is then mainly used for efficient type testing of Operations, for example in switch constructs:
///
///     switch instr.op.opcode() {
///         case .loadInt(let op):
///             doSomethingWithLoadInteger(op)
///         // ...
///         case .callFunction(let op):
///             doSomethingWithCallFunction(op)
///         // ...
///     }
///
/// This is both efficient, as only an integer value needs to be switched on, and type-safe, as it avoids type casts entirely.
enum Opcode {
    case nop(Nop)
    case loadInteger(LoadInteger)
    case loadBigInt(LoadBigInt)
    case loadFloat(LoadFloat)
    case loadString(LoadString)
    case loadBoolean(LoadBoolean)
    case loadUndefined(LoadUndefined)
    case loadNull(LoadNull)
    case loadThis(LoadThis)
    case loadArguments(LoadArguments)
    case createNamedVariable(CreateNamedVariable)
    case loadDisposableVariable(LoadDisposableVariable)
    case loadAsyncDisposableVariable(LoadAsyncDisposableVariable)
    case loadRegExp(LoadRegExp)
    case beginObjectLiteral(BeginObjectLiteral)
    case objectLiteralAddProperty(ObjectLiteralAddProperty)
    case objectLiteralAddElement(ObjectLiteralAddElement)
    case objectLiteralAddComputedProperty(ObjectLiteralAddComputedProperty)
    case objectLiteralCopyProperties(ObjectLiteralCopyProperties)
    case objectLiteralSetPrototype(ObjectLiteralSetPrototype)
    case beginObjectLiteralMethod(BeginObjectLiteralMethod)
    case endObjectLiteralMethod(EndObjectLiteralMethod)
    case beginObjectLiteralComputedMethod(BeginObjectLiteralComputedMethod)
    case endObjectLiteralComputedMethod(EndObjectLiteralComputedMethod)
    case beginObjectLiteralGetter(BeginObjectLiteralGetter)
    case endObjectLiteralGetter(EndObjectLiteralGetter)
    case beginObjectLiteralSetter(BeginObjectLiteralSetter)
    case endObjectLiteralSetter(EndObjectLiteralSetter)
    case endObjectLiteral(EndObjectLiteral)
    case beginClassDefinition(BeginClassDefinition)
    case beginClassConstructor(BeginClassConstructor)
    case endClassConstructor(EndClassConstructor)
    case classAddInstanceProperty(ClassAddInstanceProperty)
    case classAddInstanceElement(ClassAddInstanceElement)
    case classAddInstanceComputedProperty(ClassAddInstanceComputedProperty)
    case beginClassInstanceMethod(BeginClassInstanceMethod)
    case endClassInstanceMethod(EndClassInstanceMethod)
    case beginClassInstanceGetter(BeginClassInstanceGetter)
    case endClassInstanceGetter(EndClassInstanceGetter)
    case beginClassInstanceSetter(BeginClassInstanceSetter)
    case endClassInstanceSetter(EndClassInstanceSetter)
    case classAddStaticProperty(ClassAddStaticProperty)
    case classAddStaticElement(ClassAddStaticElement)
    case classAddStaticComputedProperty(ClassAddStaticComputedProperty)
    case beginClassStaticInitializer(BeginClassStaticInitializer)
    case endClassStaticInitializer(EndClassStaticInitializer)
    case beginClassStaticMethod(BeginClassStaticMethod)
    case endClassStaticMethod(EndClassStaticMethod)
    case beginClassStaticGetter(BeginClassStaticGetter)
    case endClassStaticGetter(EndClassStaticGetter)
    case beginClassStaticSetter(BeginClassStaticSetter)
    case endClassStaticSetter(EndClassStaticSetter)
    case classAddPrivateInstanceProperty(ClassAddPrivateInstanceProperty)
    case beginClassPrivateInstanceMethod(BeginClassPrivateInstanceMethod)
    case endClassPrivateInstanceMethod(EndClassPrivateInstanceMethod)
    case classAddPrivateStaticProperty(ClassAddPrivateStaticProperty)
    case beginClassPrivateStaticMethod(BeginClassPrivateStaticMethod)
    case endClassPrivateStaticMethod(EndClassPrivateStaticMethod)
    case endClassDefinition(EndClassDefinition)
    case createArray(CreateArray)
    case createIntArray(CreateIntArray)
    case createFloatArray(CreateFloatArray)
    case createArrayWithSpread(CreateArrayWithSpread)
    case createTemplateString(CreateTemplateString)
    case getProperty(GetProperty)
    case setProperty(SetProperty)
    case updateProperty(UpdateProperty)
    case deleteProperty(DeleteProperty)
    case configureProperty(ConfigureProperty)
    case getElement(GetElement)
    case setElement(SetElement)
    case updateElement(UpdateElement)
    case deleteElement(DeleteElement)
    case configureElement(ConfigureElement)
    case getComputedProperty(GetComputedProperty)
    case setComputedProperty(SetComputedProperty)
    case updateComputedProperty(UpdateComputedProperty)
    case deleteComputedProperty(DeleteComputedProperty)
    case configureComputedProperty(ConfigureComputedProperty)
    case typeOf(TypeOf)
    case void(Void_)
    case testInstanceOf(TestInstanceOf)
    case testIn(TestIn)
    case beginPlainFunction(BeginPlainFunction)
    case endPlainFunction(EndPlainFunction)
    case beginArrowFunction(BeginArrowFunction)
    case endArrowFunction(EndArrowFunction)
    case beginGeneratorFunction(BeginGeneratorFunction)
    case endGeneratorFunction(EndGeneratorFunction)
    case beginAsyncFunction(BeginAsyncFunction)
    case endAsyncFunction(EndAsyncFunction)
    case beginAsyncArrowFunction(BeginAsyncArrowFunction)
    case endAsyncArrowFunction(EndAsyncArrowFunction)
    case beginAsyncGeneratorFunction(BeginAsyncGeneratorFunction)
    case endAsyncGeneratorFunction(EndAsyncGeneratorFunction)
    case beginConstructor(BeginConstructor)
    case endConstructor(EndConstructor)
    case directive(Directive)
    case `return`(Return)
    case yield(Yield)
    case yieldEach(YieldEach)
    case await(Await)
    case callFunction(CallFunction)
    case callFunctionWithSpread(CallFunctionWithSpread)
    case construct(Construct)
    case constructWithSpread(ConstructWithSpread)
    case callMethod(CallMethod)
    case callMethodWithSpread(CallMethodWithSpread)
    case callComputedMethod(CallComputedMethod)
    case callComputedMethodWithSpread(CallComputedMethodWithSpread)
    case unaryOperation(UnaryOperation)
    case binaryOperation(BinaryOperation)
    case ternaryOperation(TernaryOperation)
    case update(Update)
    case dup(Dup)
    case reassign(Reassign)
    case destructArray(DestructArray)
    case destructArrayAndReassign(DestructArrayAndReassign)
    case destructObject(DestructObject)
    case destructObjectAndReassign(DestructObjectAndReassign)
    case compare(Compare)
    case eval(Eval)
    case beginWith(BeginWith)
    case endWith(EndWith)
    case callSuperConstructor(CallSuperConstructor)
    case callSuperMethod(CallSuperMethod)
    case getPrivateProperty(GetPrivateProperty)
    case setPrivateProperty(SetPrivateProperty)
    case updatePrivateProperty(UpdatePrivateProperty)
    case callPrivateMethod(CallPrivateMethod)
    case getSuperProperty(GetSuperProperty)
    case setSuperProperty(SetSuperProperty)
    case getComputedSuperProperty(GetComputedSuperProperty)
    case setComputedSuperProperty(SetComputedSuperProperty)
    case updateSuperProperty(UpdateSuperProperty)
    case beginIf(BeginIf)
    case beginElse(BeginElse)
    case endIf(EndIf)
    case beginWhileLoopHeader(BeginWhileLoopHeader)
    case beginWhileLoopBody(BeginWhileLoopBody)
    case endWhileLoop(EndWhileLoop)
    case beginDoWhileLoopBody(BeginDoWhileLoopBody)
    case beginDoWhileLoopHeader(BeginDoWhileLoopHeader)
    case endDoWhileLoop(EndDoWhileLoop)
    case beginForLoopInitializer(BeginForLoopInitializer)
    case beginForLoopCondition(BeginForLoopCondition)
    case beginForLoopAfterthought(BeginForLoopAfterthought)
    case beginForLoopBody(BeginForLoopBody)
    case endForLoop(EndForLoop)
    case beginForInLoop(BeginForInLoop)
    case endForInLoop(EndForInLoop)
    case beginForOfLoop(BeginForOfLoop)
    case beginForOfLoopWithDestruct(BeginForOfLoopWithDestruct)
    case endForOfLoop(EndForOfLoop)
    case beginRepeatLoop(BeginRepeatLoop)
    case endRepeatLoop(EndRepeatLoop)
    case loopBreak(LoopBreak)
    case loopContinue(LoopContinue)
    case beginTry(BeginTry)
    case beginCatch(BeginCatch)
    case beginFinally(BeginFinally)
    case endTryCatchFinally(EndTryCatchFinally)
    case throwException(ThrowException)
    case beginCodeString(BeginCodeString)
    case endCodeString(EndCodeString)
    case beginBlockStatement(BeginBlockStatement)
    case endBlockStatement(EndBlockStatement)
    case beginSwitch(BeginSwitch)
    case beginSwitchCase(BeginSwitchCase)
    case beginSwitchDefaultCase(BeginSwitchDefaultCase)
    case endSwitchCase(EndSwitchCase)
    case endSwitch(EndSwitch)
    case switchBreak(SwitchBreak)
    case loadNewTarget(LoadNewTarget)
    case print(Print)
    case explore(Explore)
    case probe(Probe)
    case fixup(Fixup)
    case beginWasmModule(BeginWasmModule)
    case endWasmModule(EndWasmModule)
    case createWasmGlobal(CreateWasmGlobal)
    case createWasmMemory(CreateWasmMemory)
    case createWasmTable(CreateWasmTable)
    case createWasmJSTag(CreateWasmJSTag)
    case createWasmTag(CreateWasmTag)
    case wrapPromising(WrapPromising)
    case wrapSuspending(WrapSuspending)
    case bindMethod(BindMethod)

    // Wasm opcodes
    case consti64(Consti64)
    case consti32(Consti32)
    case constf32(Constf32)
    case constf64(Constf64)
    case wasmReturn(WasmReturn)
    case wasmJsCall(WasmJsCall)

    // Numerical Operations
    case wasmi32CompareOp(Wasmi32CompareOp)
    case wasmi64CompareOp(Wasmi64CompareOp)
    case wasmf32CompareOp(Wasmf32CompareOp)
    case wasmf64CompareOp(Wasmf64CompareOp)
    case wasmi32EqualZero(Wasmi32EqualZero)
    case wasmi64EqualZero(Wasmi64EqualZero)
    case wasmi32BinOp(Wasmi32BinOp)
    case wasmi64BinOp(Wasmi64BinOp)
    case wasmi32UnOp(Wasmi32UnOp)
    case wasmi64UnOp(Wasmi64UnOp)
    case wasmf32BinOp(Wasmf32BinOp)
    case wasmf64BinOp(Wasmf64BinOp)
    case wasmf32UnOp(Wasmf32UnOp)
    case wasmf64UnOp(Wasmf64UnOp)

    // Numerical Conversion / Truncation operations
    case wasmWrapi64Toi32(WasmWrapi64Toi32)
    case wasmTruncatef32Toi32(WasmTruncatef32Toi32)
    case wasmTruncatef64Toi32(WasmTruncatef64Toi32)
    case wasmExtendi32Toi64(WasmExtendi32Toi64)
    case wasmTruncatef32Toi64(WasmTruncatef32Toi64)
    case wasmTruncatef64Toi64(WasmTruncatef64Toi64)
    case wasmConverti32Tof32(WasmConverti32Tof32)
    case wasmConverti64Tof32(WasmConverti64Tof32)
    case wasmDemotef64Tof32(WasmDemotef64Tof32)
    case wasmConverti32Tof64(WasmConverti32Tof64)
    case wasmConverti64Tof64(WasmConverti64Tof64)
    case wasmPromotef32Tof64(WasmPromotef32Tof64)
    case wasmReinterpretf32Asi32(WasmReinterpretf32Asi32)
    case wasmReinterpretf64Asi64(WasmReinterpretf64Asi64)
    case wasmReinterpreti32Asf32(WasmReinterpreti32Asf32)
    case wasmReinterpreti64Asf64(WasmReinterpreti64Asf64)
    case wasmSignExtend8Intoi32(WasmSignExtend8Intoi32)
    case wasmSignExtend16Intoi32(WasmSignExtend16Intoi32)
    case wasmSignExtend8Intoi64(WasmSignExtend8Intoi64)
    case wasmSignExtend16Intoi64(WasmSignExtend16Intoi64)
    case wasmSignExtend32Intoi64(WasmSignExtend32Intoi64)
    case wasmTruncateSatf32Toi32(WasmTruncateSatf32Toi32)
    case wasmTruncateSatf64Toi32(WasmTruncateSatf64Toi32)
    case wasmTruncateSatf32Toi64(WasmTruncateSatf32Toi64)
    case wasmTruncateSatf64Toi64(WasmTruncateSatf64Toi64)

    case wasmReassign(WasmReassign)
    case wasmDefineGlobal(WasmDefineGlobal)
    case wasmDefineTable(WasmDefineTable)
    case wasmDefineMemory(WasmDefineMemory)
    case wasmLoadGlobal(WasmLoadGlobal)
    case wasmStoreGlobal(WasmStoreGlobal)
    case wasmTableGet(WasmTableGet)
    case wasmTableSet(WasmTableSet)
    case wasmMemoryLoad(WasmMemoryLoad)
    case wasmMemoryStore(WasmMemoryStore)
    case beginWasmFunction(BeginWasmFunction)
    case endWasmFunction(EndWasmFunction)
    case wasmBeginBlock(WasmBeginBlock)
    case wasmEndBlock(WasmEndBlock)
    case wasmBeginLoop(WasmBeginLoop)
    case wasmEndLoop(WasmEndLoop)
    case wasmBranch(WasmBranch)
    case wasmBranchIf(WasmBranchIf)
    case wasmNop(WasmNop)
    case wasmBeginIf(WasmBeginIf)
    case wasmBeginElse(WasmBeginElse)
    case wasmEndIf(WasmEndIf)
    case wasmBeginTry(WasmBeginTry)
    case wasmBeginCatchAll(WasmBeginCatchAll)
    case wasmBeginCatch(WasmBeginCatch)
    case wasmEndTry(WasmEndTry)
    case wasmBeginTryDelegate(WasmBeginTryDelegate)
    case wasmEndTryDelegate(WasmEndTryDelegate)
    case wasmThrow(WasmThrow)
    case wasmRethrow(WasmRethrow)
    case wasmDefineTag(WasmDefineTag)

    case constSimd128(ConstSimd128)
    case wasmSimd128Compare(WasmSimd128Compare)
    case wasmSimd128IntegerUnOp(WasmSimd128IntegerUnOp)
    case wasmSimd128IntegerBinOp(WasmSimd128IntegerBinOp)
    case wasmSimd128FloatUnOp(WasmSimd128FloatUnOp)
    case wasmSimd128FloatBinOp(WasmSimd128FloatBinOp)
    case wasmI64x2Splat(WasmI64x2Splat)
    case wasmI64x2ExtractLane(WasmI64x2ExtractLane)
    case wasmSimdLoad(WasmSimdLoad)

    case wasmUnreachable(WasmUnreachable)
    case wasmSelect(WasmSelect)
}
```
&nbsp;

## Opcode Enum

&nbsp;


FuzzIL handles various **operations** under a common interface. For example:

&nbsp;

- `LoadInt` operation: loads an integer value into a variable
- `CallFunction` operation: calls a function
- `BeginWhileLoop` / `EndWhileLoop` operations: start/end of a while loop
- … (including other JavaScript/WASM-related operations)

&nbsp;

> The Opcode enum represents **all possible operations** in FuzzIL in a way that allows them to be **switched on using integers**.

&nbsp;

Each case in the enum is associated with a specific `Operation` subclass.

- For example: `.loadInteger(LoadInteger)`
    - This allows for easy categorization like `.loadInteger(...)`, `.callFunction(...)`, etc.

There is also a **1:1 mapping between opcodes and their corresponding** `Operation` **subclasses**.

```swift
switch instruction.op.opcode {
    case .loadInteger(let op):
        // `op` is of type LoadInteger
    case .callFunction(let op):
        // 'op'is of type CallFunction
    ...
}
```

&nbsp;

# Operation.swift
-----------------

&nbsp;

```swfit
// Copyright 2022 Google LLC
//
// Licensed under the Apache License, Version 2.0 (the "License");
// you may not use this file except in compliance with the License.
// You may obtain a copy of the License at
//
// https://www.apache.org/licenses/LICENSE-2.0
//
// Unless required by applicable law or agreed to in writing, software
// distributed under the License is distributed on an "AS IS" BASIS,
// WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
// See the License for the specific language governing permissions and
// limitations under the License.

/// An operation in the FuzzIL language.
///
/// Operations can be shared between different programs since they do not contain any
/// program specific data.
public class Operation {
    /// The context in which the operation can exist
    final let requiredContext: Context

    /// The context that this operations opens
    final let contextOpened: Context

    /// The attributes of this operation.
    final let attributes: Attributes

    /// The number of input variables to this operation.
    private let numInputs_: UInt16
    final var numInputs: Int {
        return Int(numInputs_)
    }

    /// The number of newly created variables in the current scope.
    private let numOutputs_: UInt16
    final var numOutputs: Int {
        return Int(numOutputs_)
    }

    /// The number of newly created variables in the inner scope if one is created.
    private let numInnerOutputs_: UInt16
    final var numInnerOutputs: Int {
        return Int(numInnerOutputs_)
    }

    /// The index of the first variadic input.
    private let firstVariadicInput_: UInt16
    final var firstVariadicInput: Int {
        assert(attributes.contains(.isVariadic))
        return Int(firstVariadicInput_)
    }

    /// The opcode for this operation.
    ///
    /// Use this to determine the type of the operation as it is significantly more efficient than type checks using the "is" or "as" operators.
    var opcode: Opcode {
        fatalError("Operations must override the opcode getter. \(self.name) does not")
    }

    init(numInputs: Int = 0, numOutputs: Int = 0, numInnerOutputs: Int = 0, firstVariadicInput: Int = -1, attributes: Attributes = [], requiredContext: Context = .empty, contextOpened: Context = .empty) {
        assert(attributes.contains(.isVariadic) == (firstVariadicInput != -1))
        assert(firstVariadicInput == -1 || firstVariadicInput <= numInputs)
        assert(contextOpened == .empty || attributes.contains(.isBlockStart))
        self.attributes = attributes
        self.requiredContext = requiredContext
        self.contextOpened = contextOpened
        self.numInputs_ = UInt16(numInputs)
        self.numOutputs_ = UInt16(numOutputs)
        self.numInnerOutputs_ = UInt16(numInnerOutputs)
        self.firstVariadicInput_ = attributes.contains(.isVariadic) ? UInt16(firstVariadicInput) : 0
    }

    /// Possible attributes of an operation.
    struct Attributes: OptionSet {
        let rawValue: UInt16

        // This operation can be mutated in a meaningful way.
        // The rough rule of thumbs is that every Operation subclass that has
        // additional members should be mutable. Example include integer values
        // (LoadInteger), string values (GetProperty and CallMethod), or Arrays
        // (CallFunctionWithSpread).
        // However, if mutations are not interesting or meaningful, or if the
        // value space is very small (e.g. a boolean), it may make sense to not
        // make the operation mutable to not degrade mutation performance (by
        // causing many meaningless mutations). An example of such an exception
        // is the isStrict member of function definitions: the value space is two
        // (true or false) and mutating the isStrict member is probably not very
        // interesting compared to mutations on other operations.
        static let isMutable                    = Attributes(rawValue: 1 << 0)

        // The operation performs a subroutine call.
        static let isCall                       = Attributes(rawValue: 1 << 1)

        // The operation is the start of a block.
        static let isBlockStart                 = Attributes(rawValue: 1 << 2)

        // The operation is the end of a block.
        static let isBlockEnd                   = Attributes(rawValue: 1 << 3)

        // The operation is used for internal purposes and should not
        // be visible to the user (e.g. appear in emitted samples).
        static let isInternal                   = Attributes(rawValue: 1 << 4)

        // The operation behaves like an (unconditional) jump. Any
        // code until the next block end is therefore dead code.
        static let isJump                       = Attributes(rawValue: 1 << 5)

        // The operation can take a variable number of inputs.
        // The firstVariadicInput contains the index of the first variadic input.
        static let isVariadic                   = Attributes(rawValue: 1 << 6)

        // This operation should occur at most once in its surrounding context.
        // If there are multiple singular operations in the same context, then
        // all but the first one are ignored (i.e. they are dead code).
        // Examples for singular operations include the default switch case or
        // a class constructor.
        // We could also fobrid having multiple singular operations in the same
        // block instead of ignoring all but the first one. However, that would
        // complicate code generation and splicing which cannot generally
        // uphold this property.
        static let isSingular                   = Attributes(rawValue: 1 << 7)

        // The operation propagates the surrounding context.
        // Most control-flow operations keep their surrounding context active.
        static let propagatesSurroundingContext = Attributes(rawValue: 1 << 8)

        // The instruction resumes the context from before its parent context.
        // This is useful for example for BeginSwitch and BeginSwitchCase.
        static let resumesSurroundingContext    = Attributes(rawValue: 1 << 9)

        // The instruction is a Nop operation.
        static let isNop                        = Attributes(rawValue: 1 << 10)

        // The instruction cannot be mutated with the input mutator
        // This is not the case for most instructions except wasm instructions where we need to
        // preserve types for correctness. Note: this is different than the .isMutable attribute.
        static let isNotInputMutable               = Attributes(rawValue: 1 << 11)
    }
}

final class Nop: Operation {
    override var opcode: Opcode { .nop(self) }

    // NOPs can have "pseudo" outputs. These should not be used by other instructions
    // and they should not be present in the lifted code, i.e. a NOP should just be
    // ignored during lifting.
    // These pseudo outputs are used to simplify some algorithms, e.g. minimization,
    // which needs to replace instructions with NOPs while keeping the variable numbers
    // contiguous. They can also serve as placeholders for future instructions.
    init(numOutputs: Int = 0) {
        // We need an empty context here as .script is default and we want to be able to minimize in every context.
        super.init(numOutputs: numOutputs, attributes: [.isNop])
    }
}


// Expose the name of an operation as instance and class variable
extension Operation {
    var name: String {
        return String(describing: type(of: self))
    }

    class var name: String {
        return String(describing: self)
    }
}
```

&nbsp;



