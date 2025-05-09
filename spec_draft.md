## SNARK-friendly abstract machine

In this text, we will describe a SNARK-friendly abstract machine.

### Objectives

1. SNARK-friendliness, including our best guess about future proof systems.
2. Relatively easy compilation from high-level languages.
3. Absence of UB (special form of attributable local non-determinism is allowed, see further).
4. Code is to be either interpreted directly, or almost trivially compiled to lower-level bytecode of a specific backend.

### Parametrization and types

Our machine will be parametrized over:

1. Base field `F`. The main field of the proof system. It also has a specified canonical bit encoding.
2. Pointer group - cyclic group, having order `PTR_ORDER`. *Note: typically, it will coincide with* `F`*, but over very small field a subgroup of multiplicative group is frequently chosen*. The size of the pointer in field elements is `PTR_WIDTH` (pointers, in theory, also might carry additional information).
3. `LIMB_BITSIZE` - canonical integers will be split into limbs of this bitsize.

It also depends on few more parameters (bounds on program size, for example), which we leave for further clarification.

We have some primitive types. They all have "canonical forms", that is, fixed representations as tuples of k elements of `F`. Type system compliance is required on the level of the code to ensure absence of undefined behavior; i.e. forming an incorrect encoding of a primitive type using operations described below is impossible.

Following primitive types are supported:

1. Base field `F`.
2. Pointer type `ptr`: represented as `[F; PTR_WIDTH]`
3. Integer types `iN`: represented as `[F; div_ceil(iN, LIMB_BITSIZE)]`

Optional extension of the system with additional fields not described here.

### Program structure overview

The structure of the program code:

1. Program is a collection of functions that may call eachother.
2. Each function is a collection of basic blocks.
3. Blocks contain instructions that operate on non-addressable frame, branching instructions and operations for interactions with RAM.

#### Frame

Each function performs transformations over a "stack frame" belonging to this particular function. We call it's elements registers. It has few functional differences to the normal stack.

First, and most important difference is that it is not addressable by pointers. All the operations that can access the stack frame only operate in terms of static offsets.

*Note: this behavior can be simulated by normal architectures by giving different address spaces to stack and heap.*

Second difference is calling convention, which largely resembles LLVM / MLIR (function can write it's outputs to any registers).

Elements of the frame are denoted `%0, %1, ...`. Additionally, we maintain another set of elements, `?1, ?2, ...`, which are called "shadow" and are unconstrained by verifier. They represent local non-deterministic advice.

#### Deterministic mode

In a deterministic mode, shadow registers are constrained as well (and assertions work differently, as described further).

#### Constants

Instructions can take a constant any time they can take a register element of a corresponding type. Some instructions also specifically require constants, these are written in `<>` brackets before the call body.

#### Question: SSA or not?

Currently, we assume that all registers are mutable and can be assigned to any amount of times. Program validator, however, does require that *all* assignments to a single register have the same type.

However, it does not seem to be a problem to also require SSA form, which would also remove this requirement.

No scoping is assumed inside of a function - any instruction can access any register.

#### Function signature

Function definition header is an expression of the form
```
define @func_name(T_0, ..., T_{k-1}; H_0, ..., H_{l-1}) -> (T'_0, ..., T'_{k'-1}; H'_0, ..., H'_{l'-1}) {
    ...
}
```

Where `T_i, H_i, T'_i, H'_i` are primitive type descriptors. Note two differences with LLVM-IR syntax: the variables are unnamed, and multiple outputs are allowed.

The function application syntax is:

```
(%a'_0, ..., %a'_{k'-1}; ?b'_0, ..., ?b'_{l'-1}) = call @func_name (%a_0, ..., %a_{k-1}; ?b_0, ..., ?b_{l-1})
```

The values `%a_0, ... %a_{k-1}; ?b'_0, ..., ?b'_{l-1}` are passed to first `0..k; 0..l` values in a stack frame of the function.

#### Function body

Body is a collection of basic blocks. Each basic block contains sequence of instructions of few kinds:

1. Branching instructions. Each block has a single branching instruction in the end, which are direct equivalents of LLVM's `br`, `switch`  and `return` instructions.
2. Elementary operations, as defined further.
3. `store` / `load` instructions.
4. Calls.
5. External calls (to be specified).

#### RAM interaction instructions

Following instructions are provided.

1. `store<const OFFSET>(ptr, F) -> ()` : stores value in `ptr + OFFSET` memory cell.
2. `load<const OFFSET>(ptr) -> F` : loads value from `ptr + OFFSET`

`store` instruction can not be applied to shadow register, and load can, and also can write to shadow registers. Essentially, that means that non-deterministic data never hits memory.

Static offsets are provided to enable simpler vectorization, if available, and to make simulation on stack machine backend easier.

#### Casts

In order to store / load an element, it must be canonicalized. I.e., there are casts

`ptr -> [F; PTR_WIDTH]`

`iN -> [F; div_ceil(N, LIMB_BITSZE)]`

These casts never fail. Reverse casts do fail if encoding is incorrect.

#### Assertions and exception handling

Two assertion instructions are provided: `assert_zero(F) -> ()` and `assert_nonzero(F) -> ()`.

*(Here be dragons):*

In a deterministic mode, all assertions (and other fail conditions) allow the prover to output `trap` value from the program.

In particular, to prove the failure (of a program known to be deterministic, or of a particular execution of non-deterministic program), it is enough for the prover to rewind a single block and execute it in a deterministic mode.

Similar approach can be used for other exception-handling, for example, out-of-gas errors.

#### Field arithmetic

Following instructions are provided:

1. `add(F, F) -> F`
2. `mul(F, F) -> F`
3. `neg(F) -> F`
4. `sub(F) -> F`
5. `inv(F) -> F` (fallible)
6. `div(F, F) -> F` (fallible)

#### Integer arithmetic

For `iN`, ring arithmetic in $\mathbb{Z}/2^N\mathbb{Z}$ operations are provided, together with `udiv`, `sdiv`, `shl` and `shr`.

Additionally, signed and unsigned integer arithmetic primitives are provided. They all have the name `(i/u)(add/sub/neg/mul/div/mod)z`, and work as follows:

Given integer inputs, attempt to perform the operation, and output the result in output `iN`, or *fail if output does not fit*.

*Note: non-wrapping operations can be compiled to this form.*

*Note: it might make sense to also support unbounded integer on the level of a dialect, which will be given provable upper bound in the instantiation.*

Bit rearrangement operation: `rearrange(iN1, ..., iNk) -> (iM1, ..., iMl)` where the sum `N1 + ... + Nk = M1 + ... + Ml`.

In particular, this simulates `zext` and unsigned truncation.

### Bitwise operations

Standard bitwise operations on `iN` are provided.

#### Pointer arithmetic

Pointer elements can be constructed from `iN`, and back (returns value in `[0..PTR_ORDER]`, fails if it does not fit in `iN`).

Pointer elements can be added, but not multiplied.