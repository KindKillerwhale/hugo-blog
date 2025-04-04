+++
title = "Fuzzilli IR Code Based Research Introduction"
description = """Perform an in-depth code-level analysis of how Fuzzilli's Intermediate Representation (IR) is implemented, \
focusing on the construction, transformation, and usage of IR within the fuzzing pipeline."""
draft = false
date = "2025-04-04"
author = "DongHyeon Hwang ( @kind_killerwhale )"
tags = ['Fuzzilli', 'IR', 'Fuzzer']
+++

&nbsp;
The `fuzzilli/Sources/Fuzzilli/FuzzIL` directory contains the following files.
&nbsp;

```sh
 
➜  FuzzIL git:(main) ✗ ls
Analyzer.swift  Code.swift     Instruction.swift  JsOperations.swift  Operation.swift  ProgramComments.swift  TypeSystem.swift  WasmOperations.swift
Blocks.swift    Context.swift  JSTyper.swift      Opcodes.swift       Program.swift    Semantics.swift        Variable.swift
 
```

&nbsp;

> This page is an introduction, and the analysis will begin in the following pages.