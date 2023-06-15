# Background 

The latest version of LLVM has dropped support for typed opaque pointers. 
This presents a unique problem for the QIR specification, which relies on 
typed opaque pointers for critical elements of the specification, e.g. 
`Qubit` and `Result` types. The utility of opaque types has been well understood - 
it provides a high level of flexibility for current and future QIR runtime implementations. Unfortunately, to stay up-to-date with the LLVM ecosystem and community, this can no longer be leveraged and we must move to other approaches. 

Here we describe one such approach that retains flexibility for `Qubit` and `Result` types but does not build upon the deprecated typed opaque pointer support in LLVM. 

# Value Semantic Types

We propose the following definition for qubits, results, and registers of qubits and results in the QIR
```llvm
%Qubit = type { i32 }
%Result = type { i32 }
%QubitArray = type { %Qubit* }
%ResultArray = type { %Result* }
```
The `%Qubit` type is a concrete struct containing a single integer member. This 
integer tracks the unique logical index of the underylying qubit in the physical 
or simulated quantum register. The intention here is that `%Qubit` instances 
are passed by value to QIR `rt` and `qis` functions and QIR implementations 
internally track the uniqueness of its index member. In doing so, we remove 
the need for this tracking to be explicit at the QIR specification layer via 
opaque pointers, and the type name is therefore retained in the QIR disassembled 
representation. We can model a measurement result similarly, with a concrete 
`%Result` struct type that encapsulates a unique identifying integer. We note that 
one could additionally add a `ptr` member to both of these types if the 
internal data tracking is desired to be more explicit, or if extra data should 
be attached to the element's representation. 

We may similarly require a typed representation for arrays of `%Qubit` and 
`%Result` elements. For this we define `%QubitArray` and `%ResultArray` concrete 
types that encapsulate a raw pointer to the underlying array of the corresponding 
element type. The pointer member of the `%QubitArray` (or the `%ResultArray`) will 
get modeled in LLVM as the raw `ptr` type, but this should only be the case 
in the full QIR reperesentation, as the current profile sub-sets typically remove the need for registers of unknown size and constant struct representations are 
propagated throughout the code for the elements of these statically known arrays. 

We propose the modification of the signature of the qubit allocation `rt` function 
and the addition of an array extraction function for qubits: 
```llvm 
declare %QubitArray @__quantum__rt__qubit_allocate_array(i64) local_unnamed_addr
declare %Qubit @__quantum__rt__qubit_extract(%QubitArray, i64) local_unnamed_addr
```
Instead of returning a raw opaque `%Array` we elect to express a concrete 
type name in the return of the qubit allocation function. The input `i64` 
argument is retained. Extraction of qubits originally was done in an opaque manner, 
with a general `@__quantum__rt__array_get_element_ptr_1d` extraction function. To 
again retain type names, we propose a specific qubit extraction function that takes 
a `%QubitArray` and an `i64` index and returns the `%Qubit` value representing 
that element in the array. 

Similar things can be done for `%Results`. 

# Concrete Example for the Full QIR

```llvm
source_filename = "LLVMDialectModule"
target datalayout = "e-m:e-p270:32:32-p271:32:32-p272:64:64-i64:64-f80:128-n8:16:32:64-S128"
target triple = "x86_64-unknown-linux-gnu"

%Qubit = type { i32 }
%Result = type { i32 }
%QubitArray = type { %Qubit* }

declare void @__quantum__rt__qubit_release_array(%QubitArray) local_unnamed_addr

declare %Result* @__quantum__qis__mz(%Qubit) local_unnamed_addr

declare void @__quantum__qis__h(%Qubit) local_unnamed_addr

declare %Qubit @__quantum__rt__qubit_extract(%QubitArray, i64) local_unnamed_addr

declare %QubitArray @__quantum__rt__qubit_allocate_array(i64) local_unnamed_addr

define void @test(i64 %N) local_unnamed_addr {
  %1 = tail call %QubitArray @__quantum__rt__qubit_allocate_array(i64 %N)
  %2 = tail call %Qubit @__quantum__rt__qubit_extract(%QubitArray %1, i64 0)
  tail call void @__quantum__qis__h(%Qubit %2)
  %3 = tail call %Result @__quantum__qis__mz(%Qubit %2)
  tail call void @__quantum__rt__qubit_release_array(%QubitArray %1)
  ret void
}

!llvm.module.flags = !{!0}

!0 = !{i32 2, !"Debug Info Version", i32 3}
```

# Concrete Example for the Base Profile
For the base profile (and other static-qubit subsets), we can take advantage 
of constant inlining of function arguments to retain a similar expressiveness 
of static circuit code as with the existing opaque pointer approach. 

```llvm
; ModuleID = '<stdin>'
source_filename = "LLVMDialectModule"
target datalayout = "e-m:e-p270:32:32-p271:32:32-p272:64:64-i64:64-f80:128-n8:16:32:64-S128"
target triple = "x86_64-unknown-linux-gnu"

%Qubit = type { i32 }
%Result = type { i32 }

declare void @__quantum__qis__h__body(%Qubit) local_unnamed_addr

declare void @__quantum__qis__cnot__body(%Qubit, %Qubit) local_unnamed_addr

declare void @__quantum__rt__result_record_output(%Result, ptr) local_unnamed_addr

declare void @__quantum__rt__array_end_record_output() local_unnamed_addr

declare void @__quantum__rt__array_start_record_output() local_unnamed_addr

declare i1 @__quantum__qis__read_result__body(%Result) local_unnamed_addr

declare void @__quantum__qis__mz__body(%Qubit, %Result) local_unnamed_addr

define void @__nvqpp__mlirgen__bell() local_unnamed_addr #0 {
  tail call void @__quantum__qis__h__body(%Qubit {i32 0})
  tail call void @__quantum__qis__cnot__body(%Qubit {i32 0}, %Qubit {i32 1})
  tail call void @__quantum__qis__mz__body(%Qubit {i32 0}, %Result {i32 0})
  %1 = tail call i1 @__quantum__qis__read_result__body(%Result {i32 0})
  tail call void @__quantum__qis__mz__body(%Qubit {i32 1}, %Result {i32 1})
  %2 = tail call i1 @__quantum__qis__read_result__body(%Result {i32 1})
  tail call void @__quantum__rt__array_start_record_output()
  tail call void @__quantum__rt__result_record_output(%Result {i32 0}, ptr null)
  tail call void @__quantum__rt__result_record_output(%Result {i32 1}, ptr null)
  tail call void @__quantum__rt__array_end_record_output()
  ret void
}

attributes #0 = { "EntryPoint" "requiredQubits"="2" "requiredResults"="2" }

!llvm.module.flags = !{!0}

!0 = !{i32 2, !"Debug Info Version", i32 3}
```