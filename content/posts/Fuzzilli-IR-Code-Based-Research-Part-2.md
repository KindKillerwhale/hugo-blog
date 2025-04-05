+++
title = "Fuzzilli IR Code Based Research Part 2 ( Opcodes.swift, Operation.swift, Program.swift, Variable.swift )"
description = """A deep dive into the core IR components of Fuzzilli, focusing on Opcodes.swift, Operation.swift, Program.swift, Variable.swift. This post is the second in the series exploring the IR internal structure of Fuzzilli.
"""
draft = false
date = "2025-04-06"
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


```swift
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

**This class serves as the abstract foundation for representing a single operation in the FuzzIL language.**


Each concrete operation (e.g., loading an integer, calling a function, beginning a for loop, etc.) inherits from this class. Subclasses implement `override var opcode` to indicate "which kind of operation this instance represents."

This design allows FuzzIL to handle operation types, input/output counts, and context-handling behaviors in a consistent and structured way.

### Summary

- The `Operation` class is the base abstraction for **the information associated with a single operation in FuzzIL**.
- Concrete operations (e.g., `LoadInteger`) inherit from this class and implement properties such as `override var opcode`.

&nbsp;

## Key Properties

&nbsp;

- `requiredContext`, `contextOpened`
    - Define how the operation **interacts with the current execution context**, and whether it **opens a new context**.
    - For example, some operations may only be valid "within a loop" or "inside a function," while others may **introduce a new context**, such as beginning a new function block.
- `attributes`
    - Encodes the **characteristics** and **constraints** of an operation using the `Attributes OptionSet`.
    - Notable flags include:
        - `.isMutable`:
            - Indicates the operation can be **mutated** during fuzzing. 
            - For example, `LoadInteger` contains internal values that can be meaningfully changed.

        - `.isCall`:
            - Marks operations that involve **function calls**.
        - `.isBlockStart` / `.isBlockEnd`:
            - Used for operations that **begin or end control-flow blocks**.
            - For example: `.isBlockStart` for `BeginWhileLoopHeader`, `.isBlockEnd` for `EndWhileLoop`.
        - `.isJump`:
            - Marks control-flow operations that may render subsequent code **unreachable**, such as `Return` or `ThrowException`.
        - `.isVariadic`:
            - Indicates the operation accepts **a variable number of inputs**. In such cases, `firstVariadicInput` specifies where the variadic segment begins.
        - `.isSingular`:
            - Enforces that the operation must be **unique within its context**, e.g., a class constructor.
        - `.isNop`:
            - Flags the operation as a **no-op**.
        - `.isNotInputMutable`:
            - Prevents the fuzzer from mutating **input variables**.
            - Especially relevant for WASM operations where **type safety** must be preserved.

- `numInputs` / `numOutputs` / `numInnerOutputs`
    - Define how many **input variables** the operation consumes and how many **output variables** it produces (some of which may be **inner-scope** only).
        - Example: LoadInteger → “0 inputs, 1 output”
        - Example: BeginFunction → “1 function object (visible externally), n parameters (scoped internally)”

- `firstVariadicInput`
    - Only valid for operations with the `.isVariadic` flag.
    - Helps distinguish between **fixed and variadic** parts of the input. For example, a function call might have fixed arguments followed by a variadic tail.
- `opcode: Opcode`
    - Returns the `Opcode` enum case representing this operation.
    - Typically implemented in subclasses as `override var opcode: Opcode { .loadInteger(self) }`, etc.
    - The presence of `fatalError` in the base class ensures that this property **must be overridden** by any concrete subclass.

## Constructor
```swift
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
```

&nbsp;


- The object is initialized with information such as the **number of input/output variables**, the index **where variadic inputs begin, attributes**, and **context constraints**.

- Internally, it uses `assert` statements to enforce invariants—for example:
    *“If the operation is variadic, then firstVariadicInput must not be -1.”*

&nbsp;

> Each Operation instance fully encapsulates information such as "the context in which it is valid", "the number of inputs and outputs", "and what attributes it possesses".

&nbsp;

## Nop

```swift
final class Nop: Operation {
    override var opcode: Opcode { .nop(self) }

    init(numOutputs: Int = 0) {
        super.init(numOutputs: numOutputs, attributes: [.isNop])
    }
}
```
&nbsp;


> The `Nop` operation represents an instruction that does nothing. Internally, its `opcode` is mapped to `.nop(self)`.

&nbsp;

It provides a default value of `numOutputs: Int = 0`, but it **can be configured to produce outputs** if necessary.

While a `Nop` **can have pseudo outputs**, these must **not be used by other instructions**, and they must **not appear in the lifted code**. Instead, they serve the following purpose:
    - During **code transformation phases** (e.g., in the Minimizer), it is useful for **temporarily disabling instructions** or **preserving structural placeholders** for future operations.
        - For example, when the Minimizer replaces a specific instruction with a `Nop` but still needs to preserve the surrounding variable index structure.

&nbsp;

The *`Operation`* class defines the **abstract common interface** for all instructions in FuzzIL. It centralizes information such as **how many variables an instruction takes or produces**, **what context it requires or opens, and whether it is eligible for mutation**.


The `Nop` operation is functionally "empty," but it plays a crucial role in **code transformation** by offering **more flexibility than simply removing instructions**.
    - For example, when the Minimizer replaces an instruction with a `Nop`, it helps preserve the **overall variable index structure**, ensuring that the program remains structurally valid and analyzable.

&nbsp;

# Program.swift
---------------


```swift
// Copyright 2019 Google LLC
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

import Foundation

/// Immutable unit of code that can, amongst others, be lifted, executed, scored, (de)serialized, and serve as basis for mutations.
///
/// A Program's code is guaranteed to have a number of static properties, as checked by code.isStaticallyValid():
/// * All input variables must have previously been defined
/// * Variables have increasing numbers starting at zero and there are no holes
/// * Variables are only used while they are visible (the block they were defined in is still active)
/// * Blocks are balanced and the opening and closing operations match (e.g. BeginIf is closed by EndIf)
/// * The outputs of an instruction are always new variables and never overwrite an existing variable
///
public final class Program {
    /// The immutable code of this program.
    public let code: Code

    /// The parent program that was used to construct this program.
    /// This is mostly only used when inspection mode is enabled to reconstruct
    /// the "history" of a program.
    public private(set) var parent: Program? = nil

    /// Comments attached to this program.
    public var comments = ProgramComments()

    /// Everything that contributed to this program. This is not preserved across protobuf serialization.
    public var contributors = Contributors()

    /// Each program has a unique ID to identify it even accross different fuzzer instances.
    public private(set) lazy var id = UUID()

    /// Constructs an empty program.
    public init() {
        self.code = Code()
        self.parent = nil
    }

    /// Constructs a program with the given code. The code must be statically valid.
    public init(with code: Code) {
        assert(code.isStaticallyValid())
        self.code = code
    }

    /// Construct a program with the given code and type information.
    public convenience init(code: Code, parent: Program? = nil, comments: ProgramComments = ProgramComments(), contributors: Contributors = Contributors()) {
        // We should never see instructions with set flags here, as Flags are currently only used temporarily (e.g. Minimizer)
        assert(code.allSatisfy { instr in instr.flags == Instruction.Flags.empty })
        self.init(with: code)
        self.comments = comments
        self.contributors = contributors
        self.parent = parent
    }

    /// The number of instructions in this program.
    public var size: Int {
        return code.count
    }

    /// Indicates whether this program is empty.
    public var isEmpty: Bool {
        return size == 0
    }

    public func clearParent() {
        parent = nil
    }

    // Create and return a deep copy of this program.
    public func copy() -> Program {
        let proto = self.asProtobuf()
        return try! Program(from: proto)
    }

    public var containsWasm: Bool {
        for instr in code {
            if instr.op is WasmOperation {
                return true
            }
        }

        return false
    }

    /// This function can be invoked from the debugger to store a program that throws an assertion to disk.
    /// The idea is to then use the FuzzILTool to lift the program, which can help debug Fuzzilli issues.
    public func storeToDisk(atPath path: String) {
        assert(path.hasSuffix(".fzil"))
        do {
            let pb = try self.asProtobuf().serializedData()
            let url = URL(fileURLWithPath: path)
            try pb.write(to: url)
        } catch {
            fatalError("Failed to serialize program to protobuf: \(error)")
        }
    }

    /// This convienience initializer helps to load a program from disk. This can be helpful for debugging purposes.
    convenience init(fromPath path: String) {
        do {
            let data = try Data(contentsOf: URL(fileURLWithPath: path))
            let pb = try Fuzzilli_Protobuf_Program(serializedBytes: data)
            try self.init(from: pb)
        } catch {
            fatalError("Failed to create program from protobuf: \(error)")
        }
    }
}

extension Program: ProtobufConvertible {
    public typealias ProtobufType = Fuzzilli_Protobuf_Program

    func asProtobuf(opCache: OperationCache? = nil) -> ProtobufType {
        return ProtobufType.with {
            $0.uuid = id.uuidData
            $0.code = code.map({ $0.asProtobuf(with: opCache) })

            if !comments.isEmpty {
                $0.comments = comments.asProtobuf()
            }

            if let parent = parent {
                $0.parent = parent.asProtobuf(opCache: opCache)
            }
        }
    }

    public func asProtobuf() -> ProtobufType {
        return asProtobuf(opCache: nil)
    }

    convenience init(from proto: ProtobufType, opCache: OperationCache? = nil) throws {
        var code = Code()
        for (i, protoInstr) in proto.code.enumerated() {
            do {
                code.append(try Instruction(from: protoInstr, with: opCache))
            } catch FuzzilliError.instructionDecodingError(let reason) {
                throw FuzzilliError.programDecodingError("could not decode instruction #\(i): \(reason)")
            }
        }

        do {
            try code.check()
        } catch FuzzilliError.codeVerificationError(let reason) {
            throw FuzzilliError.programDecodingError("decoded code is not statically valid: \(reason)")
        }

        self.init(code: code)

        if let uuid = UUID(uuidData: proto.uuid) {
            self.id = uuid
        }

        self.comments = ProgramComments(from: proto.comments)

        if proto.hasParent {
            self.parent = try Program(from: proto.parent, opCache: opCache)
        }
    }

    public convenience init(from proto: ProtobufType) throws {
        try self.init(from: proto, opCache: nil)
    }
}
```

&nbsp;

## Program Class

```swift
public final class Program {
    public let code: Code
    public private(set) var parent: Program? = nil
    public var comments = ProgramComments()
    public var contributors = Contributors()
    public private(set) lazy var id = UUID()

    public init() {
        self.code = Code()
        self.parent = nil
    }
    ...
}
```

&nbsp;

In FuzzIL, this represents a “**self-contained chunk of code**.” Internally, the core component is the `code: Code` property, which holds a **list of instructions**.

In addition, it manages metadata such as the **parent program** (`parent`), **comments** (`comments`), **contributors** (`contributors`), and a unique **identifier** (`id`).

&nbsp;

## Key Properties

- `code: Code`
    - The actual **sequence of instructions** that make up the program (stored as an array).
    - FuzzIL ensures **static validity** (e.g., proper opening and closing of variables and blocks) using `code.isStaticallyValid()`.
    - The constructor enforces this validity with `assert(code.isStaticallyValid())`.

- `parent: Program?`
    - Tracks **which program this one was derived from**.
    - For example, during fuzzing, this may point to the “original” program used in a **splice or mutation**.
    - Useful for debugging and validation—allows one to trace **how a program was generated**.

- `comments: ProgramComments`
    - Stores **annotations and additional information** attached to the program.
    - For example: “Coverage change was detected at this point,” or “An event occurred during this phase,” etc. Useful for developers and analysis tools.
- `contributors: Contributors`
    - Records **who or what contributed to generating this program**.
    - Not included in Protobuf serialization—kept locally only.
- `id: UUID`
    - A **unique identifier for the program**.
    - Ensures distinct identification during debugging or cross-session transfer.
    - Declared as `lazy`, meaning `UUID()` is only generated when the `id` property is first accessed.

&nbsp;

## Constructor & set property

```swift
public init() {
    self.code = Code()
    self.parent = nil
}
public init(with code: Code) {
    assert(code.isStaticallyValid())
    self.code = code
}
public convenience init(code: Code, parent: Program? = nil, comments: ProgramComments = ProgramComments(), contributors: Contributors = Contributors()) {
    assert(code.allSatisfy { instr in instr.flags == Instruction.Flags.empty })
    self.init(with: code)
    self.comments = comments
    self.contributors = contributors
    self.parent = parent
}
```

&nbsp;

- **Default** `init()`
    - Initializes a program with an empty `Code()` (i.e., no instructions).
    The `parent` is set to `nil`.
- `init(with code: Code)`
    - Accepts a preconstructed `Code` object (a sequence of instructions).
    - Performs validity checks using `assert`, then sets `self.code = code`.
- `init(code: Code, parent: Program?, comments: ProgramComments, contributors: Contributors = Contributors())`
    - A **convenience initializer** that allows setting `parent`, `comments`, and `contributors` in addition to `code`.
    - During initialization, it ensures that **no temporary flags are set on any of the instructions** (i.e., all flags must be empty).

&nbsp;

## Exploring Properties and Methods

```swift
public var size: Int {
    return code.count
}
public var isEmpty: Bool {
    return size == 0
}

public func clearParent() {
    parent = nil
}

public func copy() -> Program {
    let proto = self.asProtobuf()
    return try! Program(from: proto)
}

public var containsWasm: Bool {
    for instr in code {
        if instr.op is WasmOperation {
            return true
        }
    }
    return false
}


public func storeToDisk(atPath path: String) {
    assert(path.hasSuffix(".fzil"))
    do {
        let pb = try self.asProtobuf().serializedData()
        let url = URL(fileURLWithPath: path)
        try pb.write(to: url)
    } catch {
        fatalError("Failed to serialize program to protobuf: \(error)")
    }
}

convenience init(fromPath path: String) {
    do {
        let data = try Data(contentsOf: URL(fileURLWithPath: path))
        let pb = try Fuzzilli_Protobuf_Program(serializedBytes: data)
        try self.init(from: pb)
    } catch {
        fatalError("Failed to create program from protobuf: \(error)")
    }
}
```

&nbsp;

- `size` & `isEmpty`
    - `size`: Returns `code.count` — the number of instructions in the program.
    - `isEmpty`: Returns `true` if `size == 0`.
- `clearParent()`
    - Sets the `parent` to `nil`, effectively detaching the program from its ancestry.
    - Useful for avoiding memory leaks or breaking complex program trees.
- `copy()`
    - Performs a **deep copy** of the program.
    - Internally, it uses **Protobuf serialization followed by deserialization** to construct a new `Program` instance with the same content.
    - Copies most components including the code and comments.
- `containsWasm`
    - Iterates through all instructions and checks whether any operation is **WASM-specific** (`instr.op is WasmOperation`).
    - Returns `true` if at least one such instruction is found.
- `storeToDisk(atPath path: String)`
    - Serializes the program using **Protobuf** and writes it to disk at the given path (typically with a `.fzil` extension).
    - Useful for debugging or saving snapshots of the current program state.
- `init(fromPath path: String)`
    - A convenience initializer that reads a `.fzil` file from disk and deserializes it using Protobuf.
    - Enables inspection of previously saved programs during debugging using FuzzILTool.

&nbsp;

## ProtobufConvertible Extension

```swift
extension Program: ProtobufConvertible {
    public typealias ProtobufType = Fuzzilli_Protobuf_Program

    func asProtobuf(opCache: OperationCache? = nil) -> ProtobufType {
        return ProtobufType.with {
            $0.uuid = id.uuidData
            $0.code = code.map({ $0.asProtobuf(with: opCache) })

            if !comments.isEmpty {
                $0.comments = comments.asProtobuf()
            }

            if let parent = parent {
                $0.parent = parent.asProtobuf(opCache: opCache)
            }
        }
    }

    public func asProtobuf() -> ProtobufType {
        return asProtobuf(opCache: nil)
    }

    convenience init(from proto: ProtobufType, opCache: OperationCache? = nil) throws {
        var code = Code()
        for (i, protoInstr) in proto.code.enumerated() {
            do {
                code.append(try Instruction(from: protoInstr, with: opCache))
            } catch FuzzilliError.instructionDecodingError(let reason) {
                throw FuzzilliError.programDecodingError("could not decode instruction #\(i): \(reason)")
            }
        }

        do {
            try code.check()
        } catch FuzzilliError.codeVerificationError(let reason) {
            throw FuzzilliError.programDecodingError("decoded code is not statically valid: \(reason)")
        }

        self.init(code: code)

        if let uuid = UUID(uuidData: proto.uuid) {
            self.id = uuid
        }

        self.comments = ProgramComments(from: proto.comments)

        if proto.hasParent {
            self.parent = try Program(from: proto.parent, opCache: opCache)
        }
    }

    public convenience init(from proto: ProtobufType) throws {
        try self.init(from: proto, opCache: nil)
    }
}
```

&nbsp;

FuzzIL supports **serialization and deserialization of programs in Protobuf format**, allowing programs to be saved, reloaded, and used in workflows such as crash lifting.

&nbsp;

Let’s look at the flow:

&nbsp;

### Serialization Flow (`asProtobuf`)

&nbsp;

1. `func asProtobuf(opCache: OperationCache? = nil) -> Fuzzilli_Protobuf_Program`
    - Converts the entire program into a `Fuzzilli_Protobuf_Program` message, including:
        - `id` (program UUID)
        - `code` (list of instructions)
        - `comments`
        - `parent` (if any, recursively encoded)
    - Each instruction is converted via `code.map { $0.asProtobuf(...) }`.
2. `public func asProtobuf() -> ProtobufType`
    - A convenience wrapper that simply calls `asProtobuf(opCache: nil)`.
    - Performs basic serialization without a custom `OperationCache`.

&nbsp;

### Deserialization Flow (`init(from:)`)

&nbsp;

1. `convenience init(from proto: ProtobufType, opCache: OperationCache? = nil) throws`
    - Reconstructs the `code` from the Protobuf message (`Fuzzilli_Protobuf_Program`).
        - Each instruction in `proto.code` is deserialized accordingly.
    - Validates the reconstructed code with `code.check()`.
    - If valid, initializes via `self.init(code: code)`.
    - Also restores metadata:
        - `uuid` from `proto.uuid` via `UUID(uuidData:)`
        - `comments` via `ProgramComments(from: proto.comments)`
        - If `proto.hasParent` is true, recursively restores the parent via `Program(from: proto.parent, ...)`.
2. `public convenience init(from proto: ProtobufType) throws`
    - A simplified version of the above that omits the `opCache` parameter.
    - Provides a clean and easy way to decode serialized program data.

&nbsp;

# Variable.swift
```swift
// Copyright 2019 Google LLC
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

/// A variable in the FuzzIL language.
///
/// Variables names (numbers) are local to a program. Different programs
/// will have the same variable names referring to different things.
public struct Variable: Hashable, CustomStringConvertible {
    // We assume that programs will always have less than 64k variables
    private let num: UInt16?

    public init(number: Int) {
        self.num = UInt16(number)
    }

    public init() {
        self.num = nil
    }

    public var number: Int {
        return Int(num!)
    }

    public var description: String {
        return identifier
    }

    public var identifier: String {
        return "v\(number)"
    }

    public static func ==(lhs: Variable, rhs: Variable) -> Bool {
        return lhs.number == rhs.number
    }

    public static func isValidVariableNumber(_ number: Int) -> Bool {
        return UInt16(exactly: number) != nil
    }
}

extension Variable: Comparable {
    public static func <(lhs: Variable, rhs: Variable) -> Bool {
        return lhs.number < rhs.number
    }
}
```

&nbsp;

- `private let num: UInt16?`
    - Stores the variable number as a 16-bit integer (or `nil`).
    - According to the comment, it assumes variables are “less than 64k (65535)”.
    - If `num` is `nil`, it may mean the variable is in a state of “**not yet having a valid number**” (used for special purposes).

- `init(number: Int)` / `init()`
    - `init(number:)` converts the input `Int` value to `UInt16` and stores it in `num`.
        - If the conversion fails (e.g., because it’s larger than 65535), a runtime error may occur in Swift.
    - The parameterless `init()` sets `num = nil`:
        Although not commonly used in practice, it can represent an “empty variable.”
- `var number: Int`
    - Returns `num!` cast to `Int`.
    - Because it uses `!`, the assumption in actual usage is that “this variable has a valid number.”
- `identifier` and `description`
    - `identifier = "v\(number)"`
    - For example: if the `number` is 3, then `"v3"`
    - The `description` property returns the same value → When `print(variable)` is called, it appears as `"v3"`
- **Operators (Equatable, Comparable, Hashable)**
    - `==` : `lhs.number == rhs.number`
    - `<` : `lhs.number < rhs.number`
    - The Hashable protocol uses `number` to generate the hash (usually implemented together with `Equatable`)
- `isValidVariableNumber(_:)`
    - A static method that checks whether the given `Int` is in the range of `UInt16`.
    - Used to check valid range when creating a variable, i.e., to determine if it is “less than or equal to 65535.”

&nbsp;

Represents a “variable” used for input/output of `Instruction` in a FuzzIL program.

&nbsp;

For example:
```swift
let v0 = Variable(number: 0)  // "v0"
let v1 = Variable(number: 1)  // "v1"
```

&nbsp;

- `v0 == v1` is `false`, and comparisons like `v1 < v0` are also possible.
- Even if the same number (e.g., `v0`) is used in multiple programs, they may belong to different contexts.

> Therefore, the key point is that variables are **uniquely identified within a single program**.

&nbsp;

## Summary

&nbsp;

`Variable` is the **basic unit representing a single variable** in FuzzIL. Internally, it uses a 16-bit integer to represent up to 65,535 variables. This struct:

- Provides a simple identifier in the form of “**v(number)**”
- Implements `Equatable`, `Comparable`, and `Hashable`, making it usable in sets, dictionaries, sorting, etc.
- Care must be taken not to exceed the integer range, but otherwise it allows lightweight and simple variable handling

&nbsp;

>In short, this struct allows **every variable in a FuzzIL program to be identified by a single numeric index**, enabling code transformation, mutation, validation, and more to consistently manage variables in forms like “this variable is `v3`”.


# To close out this post

&nbsp;

This concludes my second post. Compared to the first one, this was relatively simple. Since I rely on translation tools and GPT to write in English, the phrasing may feel a bit awkward or stiff—I hope you’ll understand. I did my best to preserve the meaning I intended, and I hope it came across clearly despite the translation.

Thank you for reading!