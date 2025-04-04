+++
title = "Fuzzilli IR Code Based Research Part 1 ( Analyzer.swift, Blocks.swift, Context.swift )"
description = """A deep dive into the core IR components of Fuzzilli, focusing on Analyzer.swift, Blocks.swift, and Context.swift. This post kicks off a series exploring the internal structure of Fuzzilli's IR.
"""
draft = false
date = "2025-04-04"
author = "DongHyeon Hwang ( @kind_killerwhale )"
tags = ['Fuzzilli', 'IR', 'Fuzzer']
+++

&nbsp;

This is the first post in a series that deeply analyzes Fuzzilli’s IR codebase.
In this post, as indicated by the title, we’ll take a closer look at `Analyzer.swift`, `Blocks.swift`, and `Context.swift`.

# Analzer.swift
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

protocol Analyzer {
    /// Analyzes the next instruction of a program.
    ///
    /// The caller must guarantee that the instructions are given to this method in the correct order.
    mutating func analyze(_ instr: Instruction)
}

extension Analyzer {
    /// Analyze the provided program.
    mutating func analyze(_ program: Program) {
        analyze(program.code)
    }

    mutating func analyze(_ code: Code) {
        assert(code.isStaticallyValid())
        for instr in code {
            analyze(instr)
        }
    }
}

/// Determines definitions, assignments, and uses of variables.
struct DefUseAnalyzer: Analyzer {
    private var assignments = VariableMap<[Int]>()
    private var uses = VariableMap<[Int]>()
    private let code: Code
    private var analysisDone = false

    init(for program: Program) {
        self.code = program.code
    }

    mutating func finishAnalysis() {
        analysisDone = true
    }

    mutating func analyze() {
        analyze(code)
        finishAnalysis()
    }

    mutating func analyze(_ instr: Instruction) {
        assert(code[instr.index].op === instr.op)    // Must be operating on the program passed in during construction
        assert(!analysisDone)
        for v in instr.allOutputs {
            assignments[v] = [instr.index]
            uses[v] = []
        }
        for v in instr.inputs {
            assert(uses.contains(v))
            uses[v]?.append(instr.index)
            if instr.reassigns(v) {
                assignments[v]?.append(instr.index)
            }
        }
    }

    /// Returns the instruction that defines the given variable.
    func definition(of variable: Variable) -> Instruction {
        assert(assignments.contains(variable))
        return code[assignments[variable]![0]]
    }

    /// Returns all instructions that assign the given variable, including its initial definition.
    func assignments(of variable: Variable) -> [Instruction] {
        assert(assignments.contains(variable))
        return assignments[variable]!.map({ code[$0] })
    }

    /// Returns the instructions using the given variable.
    func uses(of variable: Variable) -> [Instruction] {
        assert(uses.contains(variable))
        return uses[variable]!.map({ code[$0] })
    }

    /// Returns the indices of the instructions using the given variable.
    func assignmentIndices(of variable: Variable) -> [Int] {
        assert(uses.contains(variable))
        return assignments[variable]!
    }

    /// Returns the indices of the instructions using the given variable.
    func usesIndices(of variable: Variable) -> [Int] {
        assert(uses.contains(variable))
        return uses[variable]!
    }

    /// Returns the number of instructions using the given variable.
    func numAssignments(of variable: Variable) -> Int {
        assert(assignments.contains(variable))
        return assignments[variable]!.count
    }

    /// Returns the number of instructions using the given variable.
    func numUses(of variable: Variable) -> Int {
        assert(uses.contains(variable))
        return uses[variable]!.count
    }
}

/// Keeps track of currently visible variables during program construction.
struct VariableAnalyzer: Analyzer {
    private(set) var visibleVariables = [Variable]()
    private(set) var scopes = Stack<Int>([0])

    /// Track the current maximum branch depth. Note that some wasm blocks are not valid branch targets
    // (catch, catch_all) and therefore don't contribute to the branch depth.
    private(set) var wasmBranchDepth = 0

    mutating func analyze(_ instr: Instruction) {
        // Scope management (1).
        if instr.isBlockEnd {
            assert(scopes.count > 0, "Trying to end a scope that was never started")
            let variablesInClosedScope = scopes.pop()
            visibleVariables.removeLast(variablesInClosedScope)

            if instr.op is WasmOperation {
                assert(wasmBranchDepth > 0)
                wasmBranchDepth -= 1
            }
        }

        scopes.top += instr.numOutputs
        visibleVariables.append(contentsOf: instr.outputs)

        // Scope management (2). Happens here since e.g. function definitions create a variable in the outer scope.
        // This code has to be somewhat careful since e.g. BeginElse both ends and begins a variable scope.
        if instr.isBlockStart {
            scopes.push(0)
            if instr.op is WasmOperation {
                wasmBranchDepth += 1
            }
        }

        scopes.top += instr.numInnerOutputs
        visibleVariables.append(contentsOf: instr.innerOutputs)
    }
}

/// Keeps track of the current context during program construction.
struct ContextAnalyzer: Analyzer {
    private var contextStack = Stack([Context.javascript])

    var context: Context {
        return contextStack.top
    }

    mutating func analyze(_ instr: Instruction) {
        if instr.isBlockEnd {
            contextStack.pop()
        }
        if instr.isBlockStart {
            var newContext = instr.op.contextOpened
            if instr.propagatesSurroundingContext {
                newContext.formUnion(context)
            }

            // If we resume the context analysis, we currently take the second to last context.
            // This currently only works if we have a single layer of these instructions.
            if instr.skipsSurroundingContext {
                assert(!instr.propagatesSurroundingContext)
                assert(contextStack.count >= 2)

                // Currently we only support context "skipping" for switch blocks. This logic may need to be refined if it is ever used for other constructs as well.
                assert((contextStack.top.contains(.switchBlock) && contextStack.top.subtracting(.switchBlock) == .empty))

                newContext.formUnion(contextStack.secondToTop)
            }
            
            // If we are in a loop, we don't want to propagate the switch context and vice versa. Otherwise we couldn't determine which break operation to emit.
            // TODO Make this generic for similar logic cases as well. E.g. by using a instr.op.contextClosed list.
            if (instr.op.contextOpened.contains(.switchBlock) || instr.op.contextOpened.contains(.switchCase)) {
                newContext.remove(.loop)
            } else if (instr.op.contextOpened.contains(.loop)) {
                newContext.remove(.switchBlock) 
                newContext.remove(.switchCase)
            }
            contextStack.push(newContext)
        }
    }
}

/// Determines whether code after the current instruction is dead code (i.e. can never be executed).
struct DeadCodeAnalyzer: Analyzer {
    private var depth = 0

    var currentlyInDeadCode: Bool {
        return depth != 0
    }

    mutating func analyze(_ instr: Instruction) {
        if instr.isBlockEnd && currentlyInDeadCode {
            depth -= 1
        }
        if instr.isBlockStart && currentlyInDeadCode {
            depth += 1
        }
        if instr.isJump && !currentlyInDeadCode {
            depth = 1
        }
        assert(depth >= 0)
    }
}
```

&nbsp;

## 1. Analyzer Protocol


```swift
protocol Analyzer {
    /// Analyzes the next instruction of a program.
    ///
    /// The caller must guarantee that the instructions are given to this method in the correct order.
    mutating func analyze(_ instr: Instruction)
}
```


`Analyzer` is a common interface that traverses FuzzIL code (such as instructions, programs, and code blocks) and performs specific analysis logic for each Instruction.

It analyzes one Instruction at a time through the `analyze(_ instr: Instruction)` method.

&nbsp;

```swift
extension Analyzer {
    mutating func analyze(_ program: Program) {
        analyze(program.code)
    }

    mutating func analyze(_ code: Code) {
        assert(code.isStaticallyValid())
        for instr in code {
            analyze(instr)
        }
    }
}
```

This is the default implementation (extension) of the `Analyzer` protocol.

- `analyze(program: Program)`: Traverses the entire program and calls analyze(_ code: Code) for each code block.

- `analyze(_ code: Code)`: Iterates through all Instructions in the given code in order and performs analyze(_:) on each.

- During this process, it verifies the consistency of the code in advance using `assert(code.isStaticallyValid())`.

&nbsp;

For any struct or class that conforms to the `Analyzer` protocol, only the method `func analyze(_ instr: Instruction)` needs to be implemented — the logic for traversing the entire program is automatically provided by this extension.

&nbsp;

## 2. DefUseAnalzer

```swift
struct DefUseAnalyzer: Analyzer {
    private var assignments = VariableMap<[Int]>()
    private var uses = VariableMap<[Int]>()
    private let code: Code
    private var analysisDone = false
    ...
}
```

`DefUseAnalyzer` tracks **definitions**, **reassignments**, and **uses** of **variables**.

- For example: An instruction like `v0 = LoadInt 42` marks the point where `v0` is "defined", and any subsequent instructions that "use" `v0` as an input will be recorded as uses.

&nbsp;

Key fields include:

&nbsp;

- `assignments`
    - `A VariableMap<[Int]>` that stores **a list of instruction indices where a specific variable is assigned**.
    - The first element represents the "**initial definition**", and the rest indicate "**reassignments**".

- `uses`
    - A `VariableMap<[Int]>` that tracks the list of **instruction indices where the variable is used as an input**.

- `code: Code`
    - The list of instructions to analyze.
    - This `DefUseAnalyzer` is initialized with a specific `code` block from a `Program`, and the `Instruction` indices refer to this code.

- `analysisDone`
    - A flag indicating whether the full analysis has been completed.
    - Once analysis is done, further modifications or additional analysis are disallowed.

&nbsp;

## `analze()` and `finishAnalysis()`
```swift
mutating func finishAnalysis() {
    analysisDone = true
}

mutating func analyze() {
    analyze(code)
    finishAnalysis()
}
```

- The `analyze()` function internally calls the default implementation of the `Analyzer` protocol (`analyze(code)`) and then finalizes the process with a call to `finishAnalysis()`.
- Once `finishAnalysis()` is called, it sets `analysisDone = true`, which protects the logic from any further changes after analysis has been completed.

&nbsp;

## `analze(_ instr: Instruction)`
```swift
mutating func analyze(_ instr: Instruction) {
        assert(code[instr.index].op === instr.op)    // Must be operating on the program passed in during construction
        assert(!analysisDone)
        for v in instr.allOutputs {
            assignments[v] = [instr.index]
            uses[v] = []
        }
        for v in instr.inputs {
            assert(uses.contains(v))
            uses[v]?.append(instr.index)
            if instr.reassigns(v) {
                assignments[v]?.append(instr.index)
            }
        }
    }
```

- `assert(code[instr.index].op === instr.op)`
    - Ensures that the currently analyzed `instr` is exactly the same as the instruction at the given index in `code` (i.e., the same object reference).
    - This guarantees that the analysis is inspecting the exact instruction at that point in the code.

- `assert(!analysisDone)`
    - Ensures that `analyze()` is not called again after the analysis is already finished.

- **Handling Output Variables (allOutputs)**:
    - If an instruction defines a new variable (`outputs`), then:
        - Its assignment list is initialized as `[instr.index]`
        - Its use list is initialized as an empty array `[]`

- **Handling Input Variables (inputs)**:
    - Each input variable `v` should already exist in `uses` (meaning it has been defined at least once before).
    - Since the current instruction is using `v`, the instruction index is appended to `uses[v]`.
    - If `instr.reassigns(v)` returns `true` (e.g., the variable is being reassigned in this instruction), then the instruction index is also appended to `assignments[v]`.


&nbsp;

As a result of this logic, for each variable `v`:

- The first element in `assignments[v]` indicates when it was first defined.
- Any further elements indicate reassignments.
- All elements in `uses[v]` record when it was used.

&nbsp;

## Lookup Methods
```swift
func definition(of variable: Variable) -> Instruction { ... }
func assignments(of variable: Variable) -> [Instruction] { ... }
func uses(of variable: Variable) -> [Instruction] { ... }
func assignmentIndices(of variable: Variable) -> [Int] { ... }
func usesIndices(of variable: Variable) -> [Int] { ... }
func numAssignments(of variable: Variable) -> Int { ... }
func numUses(of variable: Variable) -> Int { ... }
```

- APIs that help easily find "which instruction defined a variable" based on definition/assignment/use information such as:

    - “**Where was a variable defined?**”
    - “**Which instruction used or reassigned this variable?**”

- Internally implemented by retrieving an index array using `assignments[variable]!` or `uses[variable]!`, and returning `code[...]` through that index.

&nbsp;

## 3. VariableAnalyzer

```swift
struct VariableAnalyzer: Analyzer {
    private(set) var visibleVariables = [Variable]()
    private(set) var scopes = Stack<Int>([0])
    private(set) var wasmBranchDepth = 0

    mutating func analyze(_ instr: Instruction) { ... }
}
```

- Tracks "**which variables are visible in the current scope**", and how variables are arranged based on block-level structures (e.g., start/end of control structures).
- FuzzIL includes **block-level operations** such as `BeginIf`/`EndIf`, and variables like `innerOutputs` may only be valid within certain blocks.

&nbsp;

The fields are as follows:

&nbsp;

- `visibleVariables`
    - A list of variables that are **currently alive in the scope**
    - When a scope ends (`isBlockEnd`), variables are removed from this list

- `scopes`
    - A **stack structure** (`Stack<Int>`) that tracks how many nested scopes are active, and how many variables are created in each scope
    - Example: push a new scope at `BeginIf`, and pop it at `EndIf`.

- `wasmBranchDepth`
        - Tracks the **current branch depth** inside WASM-related blocks (e.g., loops, blocks)
        - Increments at the start of a `WasmOperation`, and decrements at the end
        - (For handling WASM block control outside of JS)

&nbsp;

## `analyze(_ instr: Instruction)`
```swift
mutating func analyze(_ instr: Instruction) {
    // Scope management (1).
    if instr.isBlockEnd {
        let variablesInClosedScope = scopes.pop()
        visibleVariables.removeLast(variablesInClosedScope)
        if instr.op is WasmOperation {
            wasmBranchDepth -= 1
        }
    }

    scopes.top += instr.numOutputs
    visibleVariables.append(contentsOf: instr.outputs)

    // Scope management (2).
    if instr.isBlockStart {
        scopes.push(0)
        if instr.op is WasmOperation {
            wasmBranchDepth += 1
        }
    }

    scopes.top += instr.numInnerOutputs
    visibleVariables.append(contentsOf: instr.innerOutputs)
}
```
&nbsp;

Let's walk through the flow:

&nbsp;

- **Block End** (`instr.isBlockEnd`)
    - Pop from the `scopes` stack → use this value (`variablesInClosedScope`) to remove that many recently added variables from `visibleVariables`
    - If it's the end of a WASM block, then `wasmBranchDepth -= 1`

- Instruction Outputs (`outputs`)
    - Increase the current scope’s variable count (`scopes.top`) by the number of output (new) variables
    - Add these new variables to `visibleVariables`

- **Block Start** (`instr.isBlockStart`)
    - Push a new scope (`push(0)`) to the stack
    - If it’s the start of a WASM block, then `wasmBranchDepth += 1`
- **innerOutputs**
    - Variables that are **not visible outside the current scope**
    - Example: `BeginFunctionDefinition` → parameters that are only visible inside the function body
    - Similarly, increase `scopes.top` by `instr.numInnerOutputs` and add those variables to `visibleVariables`

&nbsp;

> In summary, `VariableAnalyzer` keeps track of **when new variables are created**, and **when scopes open and close**, maintaining an up-to-date list of currently live variables (`visibleVariables`).

&nbsp;

## 4. ContextAnalyzer
```swift
struct ContextAnalyzer: Analyzer {
    private var contextStack = Stack([Context.javascript])

    var context: Context {
        return contextStack.top
    }

    mutating func analyze(_ instr: Instruction) { ... }
}
```

&nbsp;

> Tracks the current **execution context** of the code.
> In FuzzIL, the concept of a `Context` is used to represent various execution states—such as “inside a loop?”, “inside a switch statement?”—using a bitmask format.

&nbsp;

## `analyze(_ instr: Instruction)`
```swift
if instr.isBlockEnd {
    contextStack.pop()
}
if instr.isBlockStart {
    var newContext = instr.op.contextOpened
    if instr.propagatesSurroundingContext {
        newContext.formUnion(context)
    }
                
    if instr.skipsSurroundingContext {
        assert(!instr.propagatesSurroundingContext)
        assert(contextStack.count >= 2)

        assert((contextStack.top.contains(.switchBlock) && contextStack.top.subtracting(.switchBlock) == .empty))

        newContext.formUnion(contextStack.secondToTop)
    }     
    
    if (instr.op.contextOpened.contains(.switchBlock) || instr.op.contextOpened.contains(.switchCase)) {
        newContext.remove(.loop)
    } else if (instr.op.contextOpened.contains(.loop)) {
        newContext.remove(.switchBlock) 
        newContext.remove(.switchCase)
    }
    contextStack.push(newContext)
}
```

&nbsp;

- At block end, call `contextStack.pop()` → returns to the outer (parent) context
- At block start:
    - Retrieve `instr.op.contextOpened` to check what kind of context the new block introduces
    - If `propagatesSurroundingContext` is true, merge the new context with the surrounding (parent) context
    - If `skipsSurroundingContext` is set, adjust the logic to use the context above the current one instead
    - Finally, `push` the computed `newContext` to set it as the **current execution context**

&nbsp;

### Example Scenarios

- `BeginWhileLoop` → opens a `.loop` context, and special handling applies to instructions inside that context
- `BeginSwitch` → opens a `.switchBlock`, and each `case` adds a `.switchCase` context, etc.

&nbsp;

> This analyzer allows the system to understand things like "**Which block will be exited if a break occurs now?**"

&nbsp;

## 5. DeadCodeAnalyzer
```swift
struct DeadCodeAnalyzer: Analyzer {
    private var depth = 0

    var currentlyInDeadCode: Bool {
        return depth != 0
    }

    mutating func analyze(_ instr: Instruction) {
        if instr.isBlockEnd && currentlyInDeadCode {
            depth -= 1
        }
        if instr.isBlockStart && currentlyInDeadCode {
            depth += 1
        }
        if instr.isJump && !currentlyInDeadCode {
            depth = 1
        }
        assert(depth >= 0)
    }
}
```

&nbsp;

A simple analyzer that tracks whether the current code is **dead code** (i.e., code that will never be executed).
It uses a `depth` counter to manage whether it has entered a state where "code beyond this point will never execute" (`currentlyInDeadCode`).


&nbsp;

- If `instr.isJump && !currentlyInDeadCode`:
    - When there is a **control-flow jump** in the current block (e.g., `return`, `break`, etc.), and we were not already in dead code,
    - Then the analyzer sets `depth = 1`, marking the rest as **dead code**
- At `isBlockStart` / `isBlockEnd` points:
    - If already in dead code, it **increments or decrements** the `depth` to properly track **nested dead code blocks**.
- An assertion `assert(depth >= 0)` ensures the `depth` never drops below zero.

&nbsp;

So for patterns like ( e.g. "`if(true) { return; } // code here is dead`") the code after return is correctly marked as dead code.

In FuzzIL, `isJump` can correspond to instructions like `Return`, `Break`, `Continue`, or branching operations in JS/WASM.

&nbsp;

## **Summary**
 
- `Analyzer` **Protocol**: A common interface for walking through a FuzzIL program and analyzing each instruction
- `DefUseAnalyzer`: Tracks where variables are defined and used; useful for optimizations like dead code elimination and partial redundancy elimination
- `VariableAnalyzer`: Maintains a list of variables that are currently visible, based on block-level scoping
- `ContextAnalyzer`: Tracks the current execution context (e.g., within a loop, switch statement, etc.)
- `DeadCodeAnalyzer`: Identifies parts of the code that are no longer executed due to control-flow jumps

&nbsp;

# Blocks.swift
--------------

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

/// A block is a sequence of instruction which starts at an opening instruction (isBlockBegin is true)
/// and ends at the next closing instruction (isBlockEnd is true) of the same nesting depth.
/// An example for a block is a loop:
///
///     BeginWhileLoop
///         ...
///     EndWhileLoop
///
/// A block contains the starting and ending instructions which are also referred to as "head" and "tail".
public struct Block {
    /// Index of the head of the block
    let head: Int

    /// Index of the tail of the block group
    let tail: Int

    /// The number of instructions in this block, including the two block instructions themselves.
    public var size: Int {
        return tail - head + 1
    }

    /// Returns the indices of all instructions in this block.
    public var allInstructions: [Int] {
        return Array(head...tail)
    }

    public init(head: Int, tail: Int, in code: Code) {
        self.head = head
        self.tail = tail

        assert(head < tail)
        assert(code[head].isBlockStart)
        assert(code[tail].isBlockEnd)
        assert(code.findBlockBegin(end: tail) == head)
        assert(code.findBlockEnd(head: head) == tail)
    }

    fileprivate init(head: Int, tail: Int) {
        assert(head < tail)

        self.head = head
        self.tail = tail
    }


}

/// A block group is a sequence of blocks (and thus instructions) that is started by an opening instruction
/// that is not closing an existing block (isBlockBegin is true and isBlockEnd is false) and ends at a closing
/// instruction that doesn't open a new block (isBlockEnd is true and isBlockBegin is false).
/// An example for a block group is an if-else statement:
///
///     BeginIf
///        ; block 1
///        ...
///     BeginElse
///        ; block 2
///        ...
///     EndIf
///
public struct BlockGroup {
    /// The indices of the block instructions belonging to this block group in the code.
    private let blockInstructions: [Int]

    /// Index of the first instruction in this block group (the opening instruction).
    public var head: Int {
        return blockInstructions.first!
    }

    /// Index of the last instruction in this block group (the closing instruction).
    public var tail: Int {
        return blockInstructions.last!
    }

    /// The number of instructions in this block group.
    public var size: Int {
        return tail - head + 1
    }

    /// The number of blocks that are part of this block group.
    public var numBlocks: Int {
        return blockInstructions.count - 1
    }

    /// The indices of all block instructions belonging to this block group.
    public var blockInstructionIndices: [Int] {
        return blockInstructions
    }

    /// The indices of all instructions in this block group.
    public var instructionIndices: [Int] {
        return Array(head...tail)
    }

    /// All blocks of this block group.
    public var blocks: [Block] {
        return (0..<numBlocks).map(block)
    }

    /// Returns the i-th block in this block group.
    public func block(_ i: Int) -> Block {
        return Block(head: blockInstructions[i], tail: blockInstructions[i + 1])
    }

    /// Constructs a block group from the a list of block instructions.
    public init(_ blockInstructions: [Int], in code: Code) {
        assert(blockInstructions.count >= 2)
        self.blockInstructions = blockInstructions
        assert(code[head].isBlockGroupStart)
        assert(code[tail].isBlockGroupEnd)
        for intermediate in blockInstructions.dropFirst().dropLast() {
            assert(code[intermediate].isBlockStart && code[intermediate].isBlockEnd)
        }
    }
}
```

&nbsp;

## 1. Blocks

```swift
public struct Block {
    let head: Int
    let tail: Int
    
    public var size: Int {
        return tail - head + 1
    }

    public var allInstructions: [Int] {
        return Array(head...tail)
    }

    public init(head: Int, tail: Int, in code: Code) {
        self.head = head
        self.tail = tail

        assert(head < tail)
        assert(code[head].isBlockStart)
        assert(code[tail].isBlockEnd)
        assert(code.findBlockBegin(end: tail) == head)
        assert(code.findBlockEnd(head: head) == tail)
    }

    fileprivate init(head: Int, tail: Int) {
        assert(head < tail)

        self.head = head
        self.tail = tail
    }
}
```

&nbsp;

A `Block` represents a single FuzzIL block (e.g., a while loop, if-branch, try-catch block, etc.).
Typically, it starts with a `BeginXxx` instruction (`isBlockStart = true`) and ends with an `EndXxx` instruction (`isBlockEnd = true`).
The `head` is the instruction index where the block begins, and the `tail` is the index where it ends.

## Key Properties


- `size`
    - Calculated as `tail - head + 1`
    - Represents the **total number of instruction**s in the block (including both start and end)

- `allInstructions`
    - Returns an array covering the range `[head...tail]`
    - Lists all instruction indices that belong to this block
- **Initializer** ``init(head: Int, tail: Int, in code: Code)``
    - Performs assertions to ensure:
        - `head < tail`
        - `code[head].isBlockStart` is true
        - `code[tail].isBlockEnd` is true
    - Also verifies that:
        - `code.findBlockBegin(end: tail) == head`
        - `code.findBlockEnd(head: head) == tail`
        - These checks confirm that the block has a **properly matched begin–end pair**
- **Initializer** `fileprivate init(head: Int, tail: Int)`
    - Used when constructing a `Block` inside a `BlockGroup`
    - Only asserts `head < tail`; full validation is done in the initializer above

&nbsp;

> **Summary**: A `Block` explicitly represents the region in FuzzIL code defined by a matching `BeginXxx` and `EndXxx` instruction pair.
>  All instruction indices between `head` and `tail` are considered the **body of the block**.

&nbsp;

## 2. BlockGroup
```swift
public struct BlockGroup {
    private let blockInstructions: [Int]

    public var head: Int {
        return blockInstructions.first!
    }

    public var tail: Int {
        return blockInstructions.last!
    }

    public var size: Int {
        return tail - head + 1
    }

    public var numBlocks: Int {
        return blockInstructions.count - 1
    }

    public var blockInstructionIndices: [Int] {
        return blockInstructions
    }

    public var instructionIndices: [Int] {
        return Array(head...tail)
    }

    public var blocks: [Block] {
        return (0..<numBlocks).map(block)
    }

    public func block(_ i: Int) -> Block {
        return Block(head: blockInstructions[i], tail: blockInstructions[i + 1])
    }

    public init(_ blockInstructions: [Int], in code: Code) {
        assert(blockInstructions.count >= 2)
        self.blockInstructions = blockInstructions
        assert(code[head].isBlockGroupStart)
        assert(code[tail].isBlockGroupEnd)
        for intermediate in blockInstructions.dropFirst().dropLast() {
            assert(code[intermediate].isBlockStart && code[intermediate].isBlockEnd)
        }
    }
}
```

&nbsp;

A `BlockGroup` represents a structure where **multiple blocks are grouped together**.
A classic example of this is an **If-Else** statement.

&nbsp;

- `BeginIf` → Block 1 → `BeginElse` → Block 2 → `EndIf`
- In this case, `BeginIf`, `BeginElse`, and `EndIf` are all considered block boundaries (`isBlockStart` / `isBlockEnd`),
    so within one **BlockGroup**, there are **two Blocks**.

&nbsp;

### Why group them?


In constructs like If-Else, multiple blocks form a **single logical unit (syntactic construct)**.
It’s hard to represent such a structure using only a single `Block`.
`BlockGroup` collects the **instruction indices of all block-related instructions** (e.g., `BeginIf`, `BeginElse`, `EndIf`),
and allows each individual `Block` to be extracted and analyzed.

&nbsp;

## Key Properties

- `blockInstructions`
    - A list of **all block boundary instruction indices** within this `BlockGroup`
    - Example: In an If-Else, this might include [`BeginIf`, `BeginElse`, `EndIf`]
- `head` & `tail`
    - The first and last instruction indices of the group
    - `head = blockInstructions.first!`, `tail = blockInstructions.last!`
- `size`
    - Calculated as `tail - head + 1`
    - The total number of instructions covered by this group
- `numBlocks`
    - Computed as `blockInstructions.count - 1`
    - Indicates how many `Block`s exist in this group
    - For example: 2 for If-Else, 3 for If-ElseIf-Else
- `blocks` (Array)
    - Generated via `(0..<numBlocks).map(block)`
    - Each block is created as `Block(head: blockInstructions[i], tail: blockInstructions[i + 1])`
    - For If-Else: returns two `Blocks` like `[ (BeginIf, BeginElse), (BeginElse, EndIf) ]`
- `instructionIndices`
    - The full range `[head...tail]`
    - Represents all instruction indices covered by this BlockGroup

&nbsp;

## Constructor `init(_ blockInstructions: [Int], in code: Code)`
```swift
public init(_ blockInstructions: [Int], in code: Code) {
    assert(blockInstructions.count >= 2)
    self.blockInstructions = blockInstructions
    assert(code[head].isBlockGroupStart)
    assert(code[tail].isBlockGroupEnd)
    for intermediate in blockInstructions.dropFirst().dropLast() {
        assert(code[intermediate].isBlockStart && code[intermediate].isBlockEnd)
    }
}
```

&nbsp;

- **Precondition**: `blockInstructions.count >= 2`
    - There must be at least a start and end instruction to form a valid group
- **Validation**:
    - `code[head].isBlockGroupStart` must be true
    - `code[tail].isBlockGroupEnd` must be true
    - All intermediate elements (`dropFirst().dropLast()`) must satisfy
        `isBlockStart && isBlockEnd` (e.g., `BeginElse` or similar)
        - That is, intermediate blocks in constructs like If-Else must be **both block start and end**, representing transitions such as an `Else` block

&nbsp;

This validation ensures that **multi-block structures** (like `If`, `Else`, `ElseIf`, `Switch-case`, etc.) are **correctly grouped** into a single `BlockGroup`.

&nbsp;

## **Summary**

- `Block`: Represents a **single block** (e.g., `BeginWhileLoop` to `EndWhileLoop`)
    - Tracks the entire instruction range using `head` and `tail`
    - Provides helpers like `size` and `allInstructions` to easily access all instructions within the block
- `BlockGroup`: Represents a **multi-block structure grouped into a single construct** (e.g., If-Else, Try-Catch, Switch-Case)
    - Holds key block transition points (e.g., `BeginIf`, `BeginElse`, `EndIf`) in `blockInstructions`
    - Can reconstruct individual `Blocks` using the `blocks` property
    - `size` and `instructionIndices` allow quick access to the overall instruction range of the group

&nbsp;

> In FuzzIL, when performing **control-flow analysis or transformation**, it is crucial to identify **where each block begins and ends** and to understand **compound control structures** (e.g., `if-else-if`).
The `Block` and `BlockGroup` structures provide a clear and manageable way to handle **instruction index ranges per block**, enabling accurate analysis and rewriting of FuzzIL programs.

&nbsp;

# Context.swift
```swift
// Copyright 2020 Google LLC
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

/// Current context in the program
public struct Context: OptionSet {
    public let rawValue: UInt32

    public init(rawValue: UInt32) {
        self.rawValue = rawValue
    }

    // Default javascript context.
    public static let javascript        = Context(rawValue: 1 << 0)
    // Inside a subroutine (function, constructor, method, ...) definition.
    // This for example means that doing `return` or accessing `arguments` is allowed.
    public static let subroutine        = Context(rawValue: 1 << 1)
    // Inside a generator function definition.
    // This for example means that `yield` and `yield*` are allowed.
    public static let generatorFunction = Context(rawValue: 1 << 2)
    // Inside an async function definition.
    // This for example means that `await` is allowed.
    public static let asyncFunction     = Context(rawValue: 1 << 3)
    // Inside a method.
    // This for example means that access to `super` is allowed.
    public static let method            = Context(rawValue: 1 << 4)
    // Inside a class method.
    // This for example means that access to private properties is allowed.
    public static let classMethod       = Context(rawValue: 1 << 5)
    // Inside a loop.
    public static let loop              = Context(rawValue: 1 << 6)
    // Inside a with statement.
    public static let with              = Context(rawValue: 1 << 7)
    // Inside an object literal.
    public static let objectLiteral     = Context(rawValue: 1 << 8)
    // Inside a class definition.
    public static let classDefinition   = Context(rawValue: 1 << 9)
    // Inside a switch block.
    public static let switchBlock       = Context(rawValue: 1 << 10)
    // Inside a switch case.
    public static let switchCase        = Context(rawValue: 1 << 11)
    // Inside a wasm module
    public static let wasm              = Context(rawValue: 1 << 12)
    // Inside a function in a wasm module
    public static let wasmFunction      = Context(rawValue: 1 << 13)
    // Inside a block of a wasm function, allows branches
    public static let wasmBlock         = Context(rawValue: 1 << 14)

    public static let empty             = Context([])

    // These contexts have ValueGenerators and as such we can .buildPrefix in them.
    public var isValueBuildableContext: Bool {
        self.contains(.wasmFunction) || self.contains(.javascript)
    }

    public var inWasm: Bool {
        // .wasmBlock is propagating surrounding context
        self.contains(.wasm) || self.contains(.wasmFunction)
    }
}
```

&nbsp;

## 1. Context Structure
```swift
public struct Context: OptionSet {
    public let rawValue: UInt32

    public init(rawValue: UInt32) {
        self.rawValue = rawValue
    }

    ...
}
```

&nbsp;

A structure that adopts `OptionSet`, allowing multiple bit flags to be combined to represent a context.
Its `rawValue` is one of the `UInt32` types, enabling up to 32 independent flags.

&nbsp;

**Example**:

- If both `Context.javascript` and `Context.loop` are set, it can be expressed as:
    `someContext = [.javascript, .loop]`

&nbsp;

## Meaning of each flag


15 flag constants are defined in this code as follows.


```swift
public static let javascript        = Context(rawValue: 1 << 0)
public static let subroutine        = Context(rawValue: 1 << 1)
public static let generatorFunction = Context(rawValue: 1 << 2)
public static let asyncFunction     = Context(rawValue: 1 << 3)
public static let method            = Context(rawValue: 1 << 4)
public static let classMethod       = Context(rawValue: 1 << 5)
public static let loop              = Context(rawValue: 1 << 6)
public static let with              = Context(rawValue: 1 << 7)
public static let objectLiteral     = Context(rawValue: 1 << 8)
public static let classDefinition   = Context(rawValue: 1 << 9)
public static let switchBlock       = Context(rawValue: 1 << 10)
public static let switchCase        = Context(rawValue: 1 << 11)
public static let wasm              = Context(rawValue: 1 << 12)
public static let wasmFunction      = Context(rawValue: 1 << 13)
public static let wasmBlock         = Context(rawValue: 1 << 14)
```

&nbsp;

Let's take a look at each item one by one.

&nbsp;

- `.javascript (1 << 0)`
    - "Standard JavaScript context."
    - This means that the code is in standard JS, not in a non-JS context like WASM.
    - Features like `await` or `yield` are not allowed in this context unless `.asyncFunction` or `.generatorFunction` is also set.
- `.subroutine (1 << 1)`
    - "Inside a subroutine (e.g., function, constructor, method)."
    - This context enables constructs like `return` or access to `arguments`.
    - Set when entering any function-like block, including constructors.
- `.generatorFunction (1 << 2)`
    - "Inside a generator function (`function*`)."
    - Enables usage of generator-specific expressions like `yield` and `yield*`.
- `.asyncFunction (1 << 3)`
    - "Inside an async function (`async function`)."
    - Enables the use of `await`.
- `.method (1 << 4)`
    - "Inside an object method (not a class method)."
    - For example: `obj = { myMethod() { ... } }`.
    - Affects whether constructs like `super` are allowed.
- `.classMethod (1 << 5)`
    - "Inside a class method (instance or static)."
    - Enables features like `super` and private fields.
    - Differs from `.method` by being inside a class context.
- `.loop (1 << 6)`
    - "Inside a loop (e.g., `for`, `while`)."
    - Allows use of loop-specific control flow like `break` and `continue`.
- `.with (1 << 7)`
    - "Inside a `with` block (`with (obj) { ... }`)."
    - Though rarely used post-ES5 strict mode, still valid in JS and supported in FuzzIL.
- `.objectLiteral (1 << 8)`
    - "Inside an object literal."
    - Context for writing property definitions and shorthand method syntax.
- `.classDefinition (1 << 9)`
    - "Inside a class definition (`class X { ... }`)."
    - Enables handling of class-specific constructs like `constructor`, `static`, and private fields.
- `.switchBlock (1 << 10)`
    - "Inside a `switch` block (`switch (val) { case ... }`)."
    - Allows use of `break`, `case`, and `default`.
- `.switchCase (1 << 11)`
    - "Inside a specific `case` block within a switch."
    - For example: `case 10: ...`.
- `.wasm (1 << 12)`
    - "Inside a WebAssembly (WASM) context."
    - Indicates that the code is part of the WASM region, not JavaScript.
    - Used to differentiate during WASM code generation.
- `.wasmFunction (1 << 13)`
    - "Inside a WebAssembly function definition."
    - Enables WASM-specific operations such as indexed locals and WASM intrinsics.
- `.wasmBlock (1 << 14)`
    - "Inside a block within a WASM function (e.g., branches, loops)."
    - Enables control flow constructs specific to WASM like `br`, `br_if`, and `block`.

&nbsp;

The `public static constant 'empty = Context([])` represents a state with no active context at all.

&nbsp;

## Additional Methods & Properties

```swift
public var isValueBuildableContext: Bool {
    self.contains(.wasmFunction) || self.contains(.javascript)
}
public var inWasm: Bool {
    self.contains(.wasm) || self.contains(.wasmFunction)
}
```

&nbsp;

- `isValueBuildableContext`
    - Determines whether a `ValueGenerator` (an instruction that creates a new variable) can be used in this context.
    - This means that meaningful values can only be generated inside standard JavaScript or WASM functions (e.g., initialization, variable declarations, etc.).
- `inWasm`
    - Indicates whether the current context is related to WASM.
    - Returns `true` if either `.wasm` (the entire WASM module) or `.wasmFunction` (a specific function within the module) is set.

&nbsp;

## 4. How to Use Overview

&nbsp;

- **Context Updates**
    - Example: When a `BeginLoop` instruction appears, the `.loop` flag is turned on; it is turned off at `EndLoop`.
    - Similarly, `BeginFunction` → `.subroutine`, `BeginAsyncFunction` → `.asyncFunction | .subroutine`, etc.
    - In this way, the FuzzIL code generator or analyzer (`ContextAnalyzer`) updates the `Context` by pushing/popping or unioning/removing flags depending on the instructions (e.g., start/end of blocks).
- **Syntactic Constraints**
    - When FuzzIL generates JavaScript code, it performs checks like “only emit a `yield` if currently in a `.generatorFunction` context.”
    - Example:
    ```swift
    if context.contains(.asyncFunction) {
        // allow 'await' expression
    }
    ```
    - In this way, FuzzIL **actively enforces syntax that is only valid within specific contexts**, minimizing unnecessary exceptions.
    - **Code Optimization/Analysis**
        - Example: The Minimizer may remove a `break` statement if it is found outside a loop block — a form of context-based optimization logic.

&nbsp;

## 5. Summary

- In FuzzIL, `Context` is a **set of options expressed as a bitmask** that represents the environment (context) in which a given piece of code is executed.
- Each flag (such as `.javascript`, `.asyncFunction`, `.loop`, `.wasm`, etc.) directly corresponds to the availability of specific JavaScript/WASM syntax features (functions, loops, switch-case, etc.).
- This allows FuzzIL to **precisely control what syntax is allowed in the current context**, reducing the generation of invalid constructs and **ensuring valid code is produced**.
- For example, a `return` statement is only allowed inside a JavaScript function body (`.subroutine`), and `await` is only valid inside an async function (`.asyncFunction`) — allowing **context-aware branching**.

&nbsp;

> In summary, `Context` is a critical option set that enables FuzzIL to manage its entire code generation process in a context-sensitive manner. It ensures that various features of JavaScript and WASM can be fuzzed **safely and correctly**, always within valid syntax boundaries.



