# Lab-410 — Task 2: g++ Flags, Underlying Tools & Python Bytecode

**Course:** Compiler Construction Lab (Lab-410), Fall 2026 · **CLO1**
**Student:** Usman hassan · **Enrollment No:** f2022-003

Manually walks through each stage of g++'s pipeline, identifies the real
programs g++ calls behind the scenes, and exposes Python's hidden bytecode
compilation step.

## Part A — g++ pipeline, stage by stage (`demo.cpp`)

| Flag | Stage | Output | Size observed |
|------|-------|--------|---------------|
| `g++ -E demo.cpp -o demo.i` | Preprocessing | `demo.i` — pure text, `#include <iostream>` expanded (~907 KB); own `main()` intact at the end | 907,753 B |
| `g++ -S demo.i -o demo.s` | Compilation proper (source → assembly) | `demo.s` — x86-64 assembly (`movl`, `call`, `pushq`…) | 1,726 B |
| `g++ -c demo.s -o demo.o` | Assembling (assembly → machine code) | `demo.o` — ELF64 object file (binary); `objdump -d` disassembles it back to confirm | 2,224 B |
| `g++ demo.o -o demo` | Linking (object → executable) | `demo` — runnable; prints `Sum: 15` | 16,544 B |

## Part B — the programs g++ actually calls (`g++ -v`)

| Tool | Role | Found at |
|------|------|----------|
| `cc1plus` | The real compiler (preprocessing + compilation proper) | `/usr/libexec/gcc/x86_64-linux-gnu/13/cc1plus` — via `g++ -print-prog-name=cc1plus`; NOT on PATH |
| `as` | The assembler | `/usr/bin/as` — on PATH (`which as`) |
| `collect2` | Wrapper that calls `ld`, the linker | `/usr/libexec/gcc/x86_64-linux-gnu/13/collect2` — via `g++ -print-prog-name=collect2`; NOT on PATH |
| `ld` | The actual linker | `/usr/bin/ld` — on PATH (`which ld`) |

`as`/`ld` are public, general-purpose GNU tools (usable by any toolchain);
`cc1plus`/`collect2` are GCC-private internals, hidden in GCC's libexec dir.

## Part C — Python bytecode (`demo.py`)

- `python3 -m dis demo.py` shows the bytecode: `LOAD_FAST`, `BINARY_OP`, `RETURN_VALUE`…
  (Python 3.12 names; older versions show `BINARY_ADD`.)
- `import demo` (via `run_demo.py`) makes CPython save `__pycache__/demo.cpython-312.pyc`
  (314 B). Disassembling the `.pyc` directly yields the same bytecode — proof it is
  the compiled form, not machine code.
- The directly-run script is compiled to bytecode in memory only; only imported
  modules get a `.pyc` on disk.

## Part D — pipeline comparison

| Stage concept | C++ (g++) | Python |
|---|---|---|
| Visible intermediate text form? | Yes — `demo.i` (preprocessed source) and `demo.s` (assembly) | No — bytecode is binary/in-memory; `dis` only displays it |
| Lower-level instruction form? | Yes — assembly (`demo.s`), then machine code (`demo.o`) | Yes — bytecode (`LOAD_FAST`, `BINARY_OP`, …) |
| Separate file saved automatically? | No — intermediates need `-E`/`-S`/`-c`; plain `g++` keeps only the executable | For imported modules, yes — `__pycache__/*.pyc` is automatic |
| Who executes the final form? | The CPU, directly | The CPython virtual machine |
| Toolchain needed to re-run? | No — the linked executable runs standalone | Yes — the interpreter/VM is always required |

## Reflection (short answers)

1. `-E` → preprocessing; `-S` → compilation proper (source → assembly);
   `-c` → assembling (assembly → object/machine code); no flag (default) → linking.
2. `as`/`ld` are standalone general-purpose tools anyone can use, so they live on
   PATH; `cc1plus`/`collect2` only make sense inside g++'s pipeline, so GCC hides
   them in its private libexec directory.
3. No. Before Python code runs, CPython compiles source to bytecode (see `dis`
   output) and the VM executes that. Python does have a compilation step — it is
   automatic, in-memory, and targets a virtual instruction set, not machine code.
4. C++'s `.o`/executable holds real CPU machine code (with library code linked
   in); Python's `.pyc` holds VM bytecode — not CPU-executable, unlinked, and
   still needs the interpreter.
5. No. A `.pyc` is bytecode for the CPython VM, not machine code; the interpreter
   is still required to run it — the `.pyc` only skips the source→bytecode step
   on repeated imports.

## Files

- `demo.cpp`, `demo.s`, `demo.o` — C++ pipeline artifacts (stages 1–3 outputs)
- `demo.py`, `run_demo.py` — Python bytecode demo
- `EVIDENCE.md` — condensed terminal outputs for every step
- `build_report.py` — generates the submission docx
- `Lab410_Task2_Report_f2022-003.docx` — the lab submission document
  (delivered in chat; binary, not pushed via the text API)
