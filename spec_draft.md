### Types & parameters

#### Primitive types

`iN`. A type representing element of $\mathbb{Z}/2^N \mathbb{Z}$.

*Note: this type faithfully simulates iN type from LLVM. Optionally, we can apply stronger restrictions on N than LLVM, but this is likely unnecessary.*

`ptr`. Pointer type. Represents integers in some range `[0..ptr::MAX]`. Attempt of creating pointer out of this range fails instantly (rather than in case of dereference).

*Note: this behavior is necessary to allow backends to use native field type for dereference, without specifying it on ISA level.*

`F_N`. Field types: collection of fields. Each field type has a canonical binary representation.

### Abstract machine

Before describing the program structure, we first specify the state of the machine; but it is important to mention few things about how it operates, first:

1. It has a concept of a function call, and, therefore, a call stack.
2. The format of the data on the call stack is known at all times.

#### Stack / registers

zk ISA machine's callstack is a list of frames.

`Callstack : List<Frame>`

Each frame is just an infinite unstructured collection of typed registers.

*Note: importantly, it is impossible to create a pointer to the data living on a stack. This seems to be necessary, because otherwise the stack structure would need to be specified. It is an acceptable tradeoff: pointer dereference leads to a RAM operation in any known proof system backend anyway.*

#### Memory

A collection of memory spaces (each of size `ptr::MAX`) is specified: at least one for a primitive byte type `i8`, and for each of the supported fields.

Optionally, separate memory spaces for larger `iN`'s can be specified for more efficient vectorized stores / loads.

#### Hints

*TODO, unclear: do we have unconstrained memory? (makes fault attribution much harder). It is possibly a good choice to limit hints to pure functions (though it is ugly).*

*other option is to allow everything, but ban everything untrusted in deterministic contexts.*

#### Program structure

*TODO: Program consists of functions, each function consists of basic blocks. All of this is completely parallel to LLVM-IR, with important change being that we might not insist on SSA form of the basic block (though I think it is desirable).*

### Operations

All operations except of branching take as inputs and outputs the registers in the current stack frame.

#### Casts

1. `cast: iN -> iM` (modular reduction if M < N, append 0-s if M > N)
2. `cast: F <-> i_{F::N_BITS}` (inverse operation validates bit repr)
3. `cast_lax: iN -> F` (treats iN as unsigned, performs modular reduction if necessary)
4. `cast: iN <-> ptr` (if `2**N > ptr::MAX`, validates the conversion as prescribed by definition of `ptr`)
5. *TODO / q: casts between iN and [iK; M], likely should be allowed at least for byte repr, but maybe we can support a bit more?*

#### Arithmetic

1. `add, mul, sub, neg` are available for all ring types (i.e. iN and fields). Division is available for fields, and `sdiv` and `udiv` are available for `iN`'s. Division by zero fails.

2. `offset: (ptr, iN) -> ptr` is available. According to `ptr` requirement, out of bounds pointer produced by this operation should lead to program failure immediately and not on dereference. *Note: sometimes, backend might be able to get away with cheaper argument than rangecheck here.*

3. `shl, shr, and, or, xor` are available for `iN`.

4. `cmp` is available for all types.

#### RAM interactions

1. `load<T, addrspace>: ptr -> T`
2. `store<T, addrspace>: (ptr, T) -> ()`

#### External calls

*TODO: even if there is no precompiles, we will need them to at least interact with storage through non-deterministic hints.*

#### Branching

*TODO: specify semantics exactly*

1. Calls / returns.
2. Unconditional jump (`br`)
3. Conditional jump (`br`)
4. Switch (`switch`)
5. Phi node (`phi`), if we go SSA route (which I think is indeed justified).

static jumps by pointer are not possible, as incorrect pointer is an undefined behavior (despite LLVM having them, but these can be pretty easily simulated using switch anyway)