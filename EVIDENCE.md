# Evidence log — Lab-410 Task 2 (condensed terminal outputs)

## Part A — pipeline stages

```
$ g++ -E demo.cpp -o demo.i
$ ls -la demo.i
-rw-rw---- 1 root nogroup 907753 demo.i        # iostream fully expanded
$ tail -8 demo.i
# 2 "demo.cpp" 2

# 2 "demo.cpp"
int main() {
    int x = 5, y = 10;
    std::cout << "Sum: " << x + y << std::endl;
    return 0;
}

$ g++ -S demo.i -o demo.s
$ head -30 demo.s
        .file   "demo.cpp"
        .text
        .globl main
        .type   main, @function
main:
        endbr64
        pushq   %rbp
        movq    %rsp, %rbp
        subq    $16, %rsp
        movl    $5, -8(%rbp)
        movl    $10, -4(%rbp)
        ...
        call    _ZStlsISt11char_traitsIcEERSt13basic_ostreamIcT_ES5_PKc@PLT
        ...

$ g++ -c demo.s -o demo.o
$ objdump -d demo.o | head -12
demo.o:     file format elf64-x86-64
Disassembly of section .text:
0000000000000000 <main>:
   0:   f3 0f 1e fa          endbr64
   4:   55                   push   %rbp
   5:   48 89 e5             mov    %rsp,%rbp
   ...

$ g++ demo.o -o demo
$ ./demo
Sum: 15
```

## Part B — underlying tools

```
$ g++ -v demo.cpp -o demo 2>&1 | grep -E "cc1plus|collect2| as "
 /usr/libexec/gcc/x86_64-linux-gnu/13/cc1plus -quiet -v ... demo.cpp ... -o /tmp/ccYSveR3.s
 as -v --64 -o /tmp/cc9MiKYp.o /tmp/cc0Bd3RY.s
 /usr/libexec/gcc/x86_64-linux-gnu/13/collect2 ... -o demo ... -lstdc++ ...

$ which as; which ld
/usr/bin/as
/usr/bin/ld

$ which cc1plus; which collect2
(no output — not on PATH, as expected)

$ g++ -print-prog-name=cc1plus
/usr/libexec/gcc/x86_64-linux-gnu/13/cc1plus
$ g++ -print-prog-name=collect2
/usr/libexec/gcc/x86_64-linux-gnu/13/collect2
```

## Part C — Python bytecode

```
$ python3 -m dis demo.py
Disassembly of <code object add ...>:
  2           0 RESUME                   0
              2 LOAD_FAST                0 (x)
              4 LOAD_FAST                1 (y)
              6 BINARY_OP                0 (+)
             10 RETURN_VALUE

$ printf 'import demo\n' > run_demo.py
$ python3 run_demo.py
15
$ ls __pycache__/
demo.cpython-312.pyc        # 314 bytes — real bytecode file on disk
```

Disassembling the `.pyc` directly (via `marshal`) reproduces the identical
`LOAD_FAST / BINARY_OP / RETURN_VALUE` sequence — confirming the `.pyc`
is the compiled bytecode form.
