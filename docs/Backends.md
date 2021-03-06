## Backends in Glow

There are two directories used by backends in Glow:

1. [tools/ClassGen/Backends/](https://github.com/pytorch/glow/tree/master/tools/ClassGen/Backends):
Each backend directory here contains new
[backend-specific](#backend-specific-nodes-and-instructions-transformations)
Nodes and Instructions for the backends. If a backend provides its own
backend-specific nodes/instructions, they should be included in
[NodeGen](https://github.com/pytorch/glow/blob/master/tools/ClassGen/NodeGen.cpp)/[InstrGen](https://github.com/pytorch/glow/blob/master/tools/ClassGen/InstrGen.cpp).

2. [lib/Backends/](https://github.com/pytorch/glow/tree/master/lib/Backends): The
implementation of the backend is contained here. This includes derived classes
for [`Backend`](#backend-abstract-class) and
[`CompiledFunction`](#compiledfunction-abstract-class).

### `Backend` Abstract Class

All backends in Glow derive from the [abstract base class
`Backend`](https://github.com/pytorch/glow/blob/master/include/glow/Backends/Backend.h). There
are two pure virtual functions all backends must implement:

- `virtual std::unique_ptr<CompiledFunction> compile(Function *F, const Context &ctx) const;`

  - This function takes a `Function *F` to compile. `Context &ctx` maps the
    graph to the concrete execution environment for a specific function. It
    should return a unique pointer to the
    [`CompiledFunction`](#compiledfunction-abstract-class) of `F`. If the backend
    uses Glow low-level IR, it can call `generateAndOptimizeIR()` to generate an
    optimized `IRFunction`.

- `virtual bool isOpSupported(Kinded::Kind opKind, ElemKind elementTy) const;`

  - Returns whether the backend supports the given operation `opKind` with the
    given `ElemKind elementTy`. For example, a backend may not support a
    specific bit-width quantization kind (e.g. `Int16QTy`) at all, or may only
    support it for certain operations (e.g. `ConvolutionNodeKind`). Any
    `(opKind, elementTy)` pair passed in that returns true must be supported
    during `compile()`.

Additionally, there are virtual functions that backends can override:

- `virtual bool transformPreLowering(Function *F, CompilationMode mode) const;`

  - Allow the backend to transform the `Function *F` before [node
    lowering](https://github.com/pytorch/glow/blob/master/docs/IR.md#node-lowering)
    occurs, given some `CompilationMode mode`. For example, a backend may prefer
    to replace a ConvolutionNode followed by a ReluNode with a
    [backend-specific](https://github.com/pytorch/glow/blob/master/docs/NewBackendSpecificNode.md)
    fused ConvReluNode. This should be done prior to node lowering, as otherwise
    the ReluNode will already be lowered to a MaxNode and may be transformed by
    other optimization passes. Returns true if the Function was modified at
    all. See [below](#backend-specific-nodes-and-instructions-transformations)
    for more information.

- `virtual bool transformPostLowering(Function *F, CompilationMode mode) const;`

  - Allow the backend to transform the `Function *F` after [node
    lowering](https://github.com/pytorch/glow/blob/master/docs/IR.md#node-lowering)
    occurs, given some `CompilationMode mode`. For example, the CPU backend
    prefers to transform MaxNodes, which take a SplatNode as an input, into a
    [backend-specific](https://github.com/pytorch/glow/blob/master/docs/NewBackendSpecificNode.md)
    CPUMaxSplatNode, which takes a scalar value as a member input instead of a
    SplatNode. This should be done after node lowering, as ReluNodes are lowered
    into MaxNodes. See
    [below](#backend-specific-nodes-and-instructions-transformations) for more
    information.

- `virtual bool shouldLower(const Node *N) const;`

  - Allow the backend to prevent lowering for some `Node *N`. For example, if a
    backend supports executing a FullyConnected operator, it would want to
    prevent lowering for it and provide a backend-specific Instruction for the
    FullyConnectedNode to be
    [IRGen'd](https://github.com/pytorch/glow/blob/master/docs/IR.md#low-level-ir)
    into. Note that IRGen for a Node can be specified via the
    [ClassGen](https://github.com/pytorch/glow/blob/master/docs/ClassGen.md)
    `autoIRGen("NodeName")` call. See
    [below](#backend-specific-nodes-and-instructions-transformations) for more
    information. Returns true if `N` should be lowered.

- `virtual bool shouldShareBuffers() const;`

  - Allow the backend to disable the buffer sharing optimization. This may be
    prefered by backends which would like to do their own memory
    optimizations. Returns true by default.

- `virtual void save(Function *F, llvm::StringRef outputDir, llvm::StringRef networkName) const;`

  - Save a [standalone executable
    bundle](https://github.com/pytorch/glow/blob/master/docs/AOT.md), where the
    provided `Function *F` is compiled and then saved to `outputDir` with main
    entry name `networkName`.

### `CompiledFunction` Abstract Class

`CompiledFunction` is an abstract class that represents the result of
compilation of a `Function`. Backends must implement their own derived class
from `CompiledFunction`, which must be returned as a result of
`Backend::compile()`. `CompiledFunction` contains a single pure virtual function
that must be implemented: `virtual void execute();`. This function is
responsible for copying inputs to the device from all input
[Placeholders](https://github.com/pytorch/glow/blob/master/docs/IR.md#placeholders),
executing the function, and copying outputs back from the device to output
Placeholders. Thus after the function returns, all Placeholders for the outputs
of the function should have had their backing tensor updated.

## Backend-Specific Nodes and Instructions Transformations

Different backends may prefer to transform or optimize the graph differently for
their own specialized architecture. For example, Glow lowers ReLU down to a Max
node, taking as inputs the original tensor and a "Splat" tensor of matching
dimensions, filled with all `0`s. Glow's CPU JIT backend prefers to replace this
pattern -- a Max with a Splat input and another non-Splat input -- with a single
"CPUMaxSplat" operation that takes a scalar Splat value as input in place of an
entire Splat tensor.

### Backend-Specific Transformation

Backends have the opportunity to perform their own analysis and transformations
before or after lowering depending on their requirements. This is exposed via
`transformPreLowering()` and `transformPostLowering()` hooks, during which a
backend can transform the graph however it desires. For example, the backend
could use `transformPostLowering()` to search the graph looking for the above
`CPUMaxSplat` pattern.

#### Backend-Specific Nodes and Instructions

A backend may create its own custom Nodes and Instructions which it can insert
into the IR. This is done via [ClassGen](ClassGen.md) and included in
`tools/ClassGen/NodeGen.cpp`. For example, the CPU Backend defines `CPUMaxSplat`
in `tools/ClassGen/Backends/CPU/CPUSpecificNodes.h`:

```cpp
BB.newBackendSpecificNode("CPUMaxSplat")
    .addInput("Input")
    .addResult("Input.getType()")
    .addMember(MemberType::Float, "SplatValue")
    .setDocstring("A Max node with one splat input; CPU specific.");
```

During `transformPostLowering()`, this `CPUMaxSplat` node replaces the
aforementioned pattern. However, there must be a corresponding instruction for
this Node to be lowered to during the IRGen phase. Thus, we need a corresponding
backend-specific CPUMaxSplat instruction, defined in
`tools/ClassGen/Backends/CPU/CPUSpecificInstrs.h`:

```
BB.newBackendSpecificInstr("CPUMaxSplat")
    .addOperand("Dest", OperandKind::Out)
    .addOperand("Src", OperandKind::In)
    .addMember(MemberType::Float, "SplatValue")
    .inplaceOperand({"Dest", "Src"})
    .dataParallel()
    .autoIRGen();
```

These instructions will appear in the instruction stream sent to the CPU backend
JIT; its [standard library](JIT.md#usage-of-the-standard-library) has a kernel
for executing this `CPUMaxSplat` instruction. You can see such instructions in
the [LeNet MNIST example](Example.md#lowering-to-ir).

Note that backend-specific nodes and instructions can be treated just as any
other node or instruction defined in `tools/ClassGen/NodeGen.cpp` or
`tools/ClassGen/InstrGen.cpp`. For example, the `CPUMaxSplat` instruction
definition includes the `dataParallel()` property, allowing for data parallel
optimizations to take place.

The `tools/ClassGen/Backends/CPU/CPUSpecificNodes.h` and
`tools/ClassGen/Backends/CPU/CPUSpecificInstrs.h` files are included in
`tools/ClassGen/NodeGen.cpp` and `tools/ClassGen/InstrGen.cpp`, respectively.
