# Decaf Compiler

A full back-end compiler pipeline for the Decaf language, built in C++. Traverses an abstract syntax tree (AST) to emit TAC (Three-Address Code) instructions, translates them to MIPS assembly via a custom code generator, and runs on the SPIM simulator. Extended with live variable analysis, graph-coloring register allocation, and compiler optimizations.

---

## What It Does

- Traverses the AST using a polymorphic `Emit()` pass on each node  
- Assigns memory locations (stack vs. global segment, with offsets) to all variables, parameters, and temporaries  
- Emits TAC instructions for all Decaf constructs and translates them to MIPS assembly  

---

## Supported Language Features

| Feature | Details |
|--------|---------|
| Arithmetic & logic | All operators; missing ones simulated from TAC primitives |
| Control flow | if, while, for, break with correct label/branch generation |
| Functions | Parameter passing, stack frame setup, main as SPIM entry point |
| Arrays | Heap allocation via `NewArray`, bounds checking at runtime, `length()` accessor |
| Classes & objects | Instance variable layout, vtable construction, dynamic dispatch |
| Built-ins | Print, ReadInteger, ReadLine, StringEqual, Alloc, Halt |
| Runtime errors | Array size ≤ 0, out-of-bounds subscript |
| Linker errors | Missing main definition |

---

## Architecture


Decaf source
└── Scanner (Flex) + Parser (Yacc)
└── AST construction
└── Emit() traversal → TAC instruction list
└── Mips::Emit() → MIPS assembly
└── SPIM execution


---

## Register Allocator

The default MIPS backend only uses 2 registers and spills everything to memory. This pass replaces that with a full graph-coloring allocator:

### Live Variable Analysis

Constructs the control flow graph (CFG) for each function and runs iterative dataflow analysis:


OUT[n] = ∪ IN[succ(n)]
IN[n] = GEN[n] ∪ (OUT[n] − KILL[n])


Iterated to fixed point to compute live variable sets at every program point.

### Interference Graph

For each instruction, variables defined (KILL) that are live-out (OUT) interfere with each other. An edge is added between each such pair in the interference graph.

### Graph Coloring (Chaitin's Algorithm)

- Repeatedly remove nodes with degree < k (k = number of general-purpose registers)  
- Recursively color the subgraph  
- Assign a color (register) different from all neighbors on the way back  
- If no node has degree < k: spill the highest-degree node (no register assigned)  
- Caller-save policy is used: all live variables are saved/restored around function calls  

---

## Optimization Passes

Optional optimization passes implemented before code generation:

- Dead Code Elimination — removes instructions whose results are never used  
- Constant Folding — evaluates constant expressions at compile time  
- Constant Propagation — replaces variable uses with known constant values  
- Copy Propagation — eliminates redundant copies  
- Common Subexpression Elimination — reuses previously computed values  

---

## Performance

Performance is measured using a weighted cycle model on SPIM:


cost = (#reads × 10) + (#writes × 10) + (#branches × 2) + (#other × 1)


The register allocator targets keeping performance cost within 20% of a reference solution. Optimizations are run iteratively since one pass can unlock further opportunities in another.

---

Interested in learning more or discussing the implementation? Feel free to reach out at miparikh@umich.edu
