As I was reading Chris Seaton’s dissertation, I was thinking about
compilation strategies for late-bound languages.  I think there’s a
straightforward ahead-of-time compilation strategy that would maybe
give a significant fraction of optimal (CPU) performance, maybe 30%,
while retaining full dynamic-dispatch semantics.

Experience from Ur-Scheme
-------------------------

My toy compiler Ur-Scheme got surprisingly good performance.  About
20% of GCC on the dumb fibonacci microbenchmark at the time (though
see below about how GCC has improved!), and about 250% of Chicken’s
performance on itself compiling itself, i.e., when compiled with
Chicken, it took 2½ times as long to compile itself as when compiled
with itself.  One technique it used for this was to open-code
primitive operations following a type check.  For example, each call
to the `string-length` function was converted into the following code:

        # string-length inlined primitive
        call ensure_string
        lea 8(%eax), %ebx
        push %ebx
        movl 4(%eax), %eax
        pop %ebx
        sal %eax
        sal %eax
        inc %eax

This was generated by this Scheme code:

    (define (inline-string-length nargs)
      (assert-equal 1 nargs)
      (comment "string-length inlined primitive")
      (extract-string)
      (asm-pop ebx)
      (native-to-scheme-integer tos))

The `ensure_string` millicode routine could also have been open-coded
and perhaps should have:

    ensure_string:
            # test whether %eax has magic: 0xbabb1e
            # first, ensure that it's a pointer, not something unboxed
            test $3, %eax
            jnz k_2
            # now, test its magic number
            cmpl $0xbabb1e, (%eax)
            jnz k_2
            ret

But on modern CPUs, this is a fetch, four ops after macro-op fusion,
and no mispredicted branches or cache misses, so it’s not extremely
expensive.  In Ur-Scheme, the handling of failed type checks like
these is nasty, brutal, and short; the program just spews out an error
to stderr and peremptorily exits.

            .section .rodata
            # align pointers so they end in binary 00
            .align 4
    _stringP_4:
            .long 0xbabb1e
            .long 20
            .ascii "error: not a string\n"
            .text
    k_2:
            movl $_stringP_4, %eax
            jmp report_error
    report_error:
            call ensure_string
            lea 8(%eax), %ebx
            push %ebx
            movl 4(%eax), %eax
            # fd 2: stderr
            movl $2, %ebx
            movl %eax, %edx
            pop %ecx
            movl $4, %eax
            int $0x80
            movl $1, %ebx
            movl $1, %eax
            int $0x80

But, at the point that this happens, *the address of the failing code
is on the stack*.  You could look at it and see what operation it was
trying to apply, then fall back to a generic method dispatch, then
jump back to where the result from `string-length` was expected.

If you open-code the check, such fallbacks would be a little more
complicated, because i386 and amd64 don’t have the conditional call
instructions their predecessor the 8080 had, so you can't just
blithely jump to `k_2` — you’ll never get back!  You’d have to jump to
a trampoline generated for just that callsite, which, as we’ll see
later, is what SBCL and OCaml do.

In some cases I *did* open-code the dynamic type checks.  The code
`(char->integer #\0)`, for example, gets compiled as follows:

        push %eax
        movl $2 + 48<<2, %eax
        test $1, %eax
        jnz not_a_character
        test $2, %eax
        je not_a_character
        cmpl $2 + 256<<2, %eax
        jnb not_a_character
        dec %eax

This code is obviously kind of stupid; among other things, we know
`#\0` is a character because it's a compile-time character constant,
and furthermore the whole expression should just constant-fold to
something like `movl $193, %eax`.

So it surprised me that this kind of nonsense was still faster than
what most Scheme compilers were doing: the extra 5 or 6 (!)
instructions and the millicode calls and returns are still faster!
However, Chez Scheme has been open-sourced since then, and both it and
probably Ikarus likely generate higher-quality code.

SBCL: almost twice as fast as Ur-Scheme, before adding declarations
-------------------------------------------------------------------

SBCL *does* do the whole thing I was saying above where you could
open-code your type check followed by open-coding the primitive
operation.  Here we see how it handles `cdr` on what it hopes is a
list:

    This is SBCL 1.0.57.0.debian, an implementation of ANSI Common Lisp.
    ...
    * (defun mylen (lst) (if (null lst) 0 (mylen (cdr lst))))

    MYLEN
    * (mylen '(3 5 1))

    0
    * (mylen '(3 5 . 1))

    debugger invoked on a TYPE-ERROR in thread
    #<THREAD "main thread" RUNNING {1002978D43}>:
      The value 1 is not of type LIST.
    ...
    (MYLEN 1)
    0] 0

So when we invoked it on an improper list, `cdr` crashed.  What does
the underlying code look like?

    * (disassemble 'mylen)

    ; disassembly for MYLEN
    ; 029B5D48:       4881FE17001020   CMP RSI, 537919511         ; no-arg-parsing entry point
    ;       4F:       7508             JNE L0

That ridiculous number, 0x20100017, is NIL.  This is what the `(if
(null lst) ...)` compiled to.

    ;       51:       31D2             XOR EDX, EDX
    ;       53:       488BE5           MOV RSP, RBP
    ;       56:       F8               CLC
    ;       57:       5D               POP RBP
    ;       58:       C3               RET

That’s “return 0”.  In SBCL the integer type-tag is 0 in the low bit,
so 0 as an integer is really 0 (in `EDX`), and in SBCL’s bizarre
calling convention, the carry flag needs to be clear to indicate that
this isn’t a multiple-value return.

Now we are going to see what `cdr` looks like.  First we check to see
if the argument is a list:

    ;       59: L0:   8BC6             MOV EAX, ESI
    ;       5B:       240F             AND AL, 15
    ;       5D:       3C07             CMP AL, 7
    ;       5F:       751E             JNE L1

So that’s the open-coded type test, four instructions, then a
predicted-not-taken branch to an error-handling trampoline welded onto
the end of the function.  When there are multiple such checks in a
function, each check gets its own trampoline.  So now we have the
open-coded implementation of `cdr` itself:

    ;       61:       488BC6           MOV RAX, RSI
    ;       64:       488B5001         MOV RDX, [RAX+1]

That’s it, that’s all of `cdr`.  Actually half of that was just a
totally unnecessary register move.  So then all that's left is the
function’s tail call:

    ;       68:       488B0581FFFFFF   MOV RAX, [RIP-127]         ; #<FDEFINITION object for MYLEN>
    ;       6F:       B902000000       MOV ECX, 2
    ;       74:       FF7508           PUSH QWORD PTR [RBP+8]
    ;       77:       FF6009           JMP QWORD PTR [RAX+9]

Then we have two error trampolines on the end of the function, the
first of which has no visible callers:

    ;       7A:       CC0A             BREAK 10                   ; error trap
    ;       7C:       02               BYTE #X02
    ;       7D:       18               BYTE #X18                  ; INVALID-ARG-COUNT-ERROR
    ;       7E:       54               BYTE #X54                  ; RCX

But the other is the one we invoked above:

    ;       7F: L1:   CC0A             BREAK 10                   ; error trap
    ;       81:       04               BYTE #X04
    ;       82:       02               BYTE #X02                  ; OBJECT-NOT-LIST-ERROR
    ;       83:       FE9501           BYTE #XFE, #X95, #X01      ; RSI

The trap instruction here is followed by five bytes that the trap
handler presumably can load, by indexing off the program counter the
BREAK saves on the stack, and interpret as it wishes.

Perhaps a more illuminating function for these purposes would be
something that conses, because when it conses it has to have an
“exception handler” for the nursery being full, in which case it
beseeches the garbage collector for memory, similar to falling back to
generic method dispatch.  We’ll just wrap the built-in `cons`
function.

    * (defun mycons (a d) (cons a d))

    MYCONS
    * (cons 'x '(30 2))

    (X 30 2)
    * (disassemble 'mycons)

    ; disassembly for MYCONS
    ; 02A2D7E8:       49896C2440       MOV [R12+64], RBP          ; no-arg-parsing entry point

That instruction has to do with SBCL’s “pseudo-atomic” handling of
interrupts during memory allocation.  I think R12 is where SBCL keeps
a pointer to thread-local data ([from a note on Steve Losh’s
blog](https://stevelosh.com/blog/2018/05/fun-with-macros-gathering/)
and [a note on Paul Khuong’s
blog](https://pvk.ca/Blog/LowLevel/VM_tricks_safepoints.html)) and
that [R12+64] in particular is `thread.pseudo-atomic-bits`.

Next we load the (thread-local!) allocation pointer
`thread.alloc-region` into L11 and bump it by 16 bytes:

    ;      7ED:       4D8B5C2418       MOV R11, [R12+24]
    ;      7F2:       498D4B10         LEA RCX, [R11+16]

But now we need to see if the new candidate allocation pointer is
still within the nursery:

    ;      7F6:       49394C2420       CMP [R12+32], RCX
    ;      7FB:       7628             JBE L2

If not, we jump off the fast path to the garbage-collection trampoline
at L2; but normally we continue on the fast path:

    ;      7FD:       49894C2418       MOV [R12+24], RCX
    ;      802:       498D4B07         LEA RCX, [R11+7]

So now we have our newly allocated dotted pair; it cost us six
instructions, plus the following three instructions for deferred
interrupt handling:

    ;      806: L0:   49316C2440       XOR [R12+64], RBP
    ;      80B:       7402             JEQ L1
    ;      80D:       CC09             BREAK 9                    ; pending interrupt trap

The L0 there is where the slow path rejoins the main flow of the
program.  Normally we expect the value at [R12+64] to be unchanged
from when we stored RBP there earlier, but if not, we handle the
pending interrupt here.

Now that we’re done with the open-coded memory allocation, all that’s
left is the open-coded initialization of `cons` itself and then the
function epilogue:

    ;      80F: L1:   488959F9         MOV [RCX-7], RBX
    ;      813:       48897901         MOV [RCX+1], RDI
    ;      817:       488BD1           MOV RDX, RCX
    ;      81A:       488BE5           MOV RSP, RBP
    ;      81D:       F8               CLC
    ;      81E:       5D               POP RBP
    ;      81F:       C3               RET

And then we have the exception handlers that are welded onto the end
of the function:

    ;      820:       CC0A             BREAK 10                   ; error trap
    ;      822:       02               BYTE #X02
    ;      823:       18               BYTE #X18                  ; INVALID-ARG-COUNT-ERROR
    ;      824:       54               BYTE #X54                  ; RCX

And here’s the one we came for, beseeching the garbage collector for
16 precious bytes for our `cons`:

    ;      825: L2:   6A10             PUSH 16
    ;      827:       4C8D1C2570724200 LEA R11, [#x427270]        ; alloc_tramp
    ;      82F:       41FFD3           CALL R11
    ;      832:       59               POP RCX
    ;      833:       488D4907         LEA RCX, [RCX+7]
    ;      837:       EBCD             JMP L0

Although SBCL doesn’t, you could use precisely the same approach, with
the inlined primitive fast path and falling back to a slow path, for
generic method dispatch.

(Above I’ve said that I think the allocation pointer is thread-local,
but I haven’t observed wonderful scalability running allocation-heavy
code in multiple SBCL threads.)

SBCL does about 57 million dumbfib leaf calls per second on this
laptop without declarations, dispatching all its arithmetic through
generic functions:

    * (defun dumbfib (x) (if (< x 2) 1 (+ (dumbfib (1- x)) (dumbfib (1- (1- x))))))

    DUMBFIB
    * (mapcar #'dumbfib '(0 1 2 3 4 5 6 7))

    (1 1 2 3 5 8 13 21)
    * (time (dumbfib 35))

    Evaluation took:
      0.263 seconds of real time
      0.264016 seconds of total run time (0.264016 user, 0.000000 system)
      100.38% CPU
      734,384,636 processor cycles
      0 bytes consed

    14930352

### The surprising comparison with GCC (11× as fast) and Ur-Scheme (half as fast as SBCL) ###

GCC by comparison does about 650 million leaf calls per second, 11
times as fast.

    $ cat fib.c
    /* ... */
    __attribute__((fastcall)) int fib(int n)
    {
        return n < 2 ? 1 : fib(n-1) + fib(n-2);
    }
    main(int c, char **v) { printf("%d\n", fib(atoi(v[1]))); }
    $ gcc -O3 fib.c -o fib
    fib.c:7:1: warning: ‘fastcall’ attribute ignored [-Wattributes]
    fib.c: In function ‘main’:
    fib.c:10:25: warning: incompatible implicit declaration of built-in function ‘printf’ [enabled by default]

(The compiler is warning us here that `fastcall` here is not relevant
on amd64 — it gives the compiler permission not to use the shitty
inefficient calling convention specified by the standard i386 ABI.)

    : user@debian:~/devel/dev3; time ./fib 40
    165580141

    real    0m0.254s
    user    0m0.252s
    sys     0m0.000s

However, comparing Ur-Scheme, I got a nasty surprise!  Ur-Scheme’s
code presumably is the same as when I wrote it 13 years ago in 02008,
but now it SUCKS compared to GCC — it’s even slower than SBCL!

    $ cat fib.scm
    ;; Dumb Fibonacci picobenchmark 
    (define (fib n) (if (< n 2) 1 (+ (fib (1- n)) (fib (1- (1- n))))))
    (display (number->string (fib 35))) (newline)
    $ ./urscheme-compiler < fib.scm > fib.s
    : user@debian:~/devel/urscheme; gcc fib.s -o dumbfib
    fib.s: Assembler messages:
    fib.s:5: Error: operand type mismatch for `push'
    ...
    $ gcc -m32 fib.s -o dumbfib
    $ time ./dumbfib
    14930352

    real    0m0.661s
    user    0m0.552s
    sys     0m0.000s

That’s only 27 million leaf calls per (user) second.  Now instead of
20 *percent* of GCC’s performance, it’s getting one *twentieth*, or
*five* percent.  Actually *four* percent.  I’m guessing that the
shitty code it emits is a lot worse at exploiting modern superscalar
OoO processors because it’s unceasingly full of dependencies.  The 20%
number was on my 700MHz Pentium III!  Valgrind says it runs
“2,851,829,048” instructions, so that’s about two instructions per
cycle.

It’s clear that I don’t know much and should spend some time
taking measurements!

Hmm, it looks like GCC went pretty hard in on the optimization
here... the above one-line C `fib` function compiled to 169
instructions.  I think GCC inlined it into itself 13 times!  But
`-fno-inline` only makes it slightly slower:

    $ gcc -fno-inline -O3 fib.c -o fib
    fib.c:7:1: warning: ‘fastcall’ attribute ignored [-Wattributes]
    fib.c: In function ‘main’:
    fib.c:10:25: warning: incompatible implicit declaration of built-in function ‘printf’ [enabled by default]
    $ time ./fib 40
    165580141

    real    0m0.390s
    user    0m0.388s
    sys     0m0.000s

I mean that’s still 430 million leaf calls per second, and some of
those are real calls, although GCC still seems to have removed half of
the recursion by realizing that integer addition is associative:

      400570:       55                      push   %rbp
      400571:       53                      push   %rbx
      400572:       89 fb                   mov    %edi,%ebx            # argument n is in rdi (...rsi, rdx, rcx...)
      400574:       48 83 ec 08             sub    $0x8,%rsp
      400578:       83 ff 01                cmp    $0x1,%edi            # n < 2?
      40057b:       7e 1f                   jle    40059c <fib+0x2c>    # otherwise:
      40057d:       31 ed                   xor    %ebp,%ebp            # initialize loop accumulator to 0
      40057f:       90                      nop
      400580:       8d 7b ff                lea    -0x1(%rbx),%edi      # edi ← n-1 (set up argument)
      400583:       83 eb 02                sub    $0x2,%ebx            # ebx ← n-2 (callee-saved!)
      400586:       e8 e5 ff ff ff          callq  400570 <fib>         # (recursive call)
      40058b:       01 c5                   add    %eax,%ebp            # add return value to accumulator
      40058d:       83 fb 01                cmp    $0x1,%ebx            # loop termination test
      400590:       7f ee                   jg     400580 <fib+0x10>    # loop back to the lea
      400592:       8d 45 01                lea    0x1(%rbp),%eax       # add one final 1 to the accumulator for return
      400595:       48 83 c4 08             add    $0x8,%rsp            # function epilogue
      400599:       5b                      pop    %rbx
      40059a:       5d                      pop    %rbp
      40059b:       c3                      retq   
      40059c:       b8 01 00 00 00          mov    $0x1,%eax            # base case almost never aken
      4005a1:       eb f2                   jmp    400595 <fib+0x25>

OCaml allocation
----------------

I wrote this code in OCaml:

    let rec nlist n cdr = if n = 0 then cdr else nlist (n-1) (n::cdr)
    let rec mnlist m n = if m = 0 then [] else (ignore (nlist n []); mnlist (m-1) n)
    let m = 2000*1000 and n = 500 ;;

    print_endline ("m=" ^ (string_of_int m) ^ " n=" ^ (string_of_int n)) ;
    mnlist m n

This ran in about 2.3 seconds, thus consing a billion list nodes in
2.3 nanoseconds each.  Here’s the disassembled machine code of
`nlist`:

    Dump of assembler code for function camlTimealloc2__nlist_1030:
       0x0000000000403530 <+0>:     sub    $0x8,%rsp
       0x0000000000403534 <+4>:     mov    %rax,%rsi
       0x0000000000403537 <+7>:     cmp    $0x1,%rsi
       0x000000000040353b <+11>:    jne    0x403548 <camlTimealloc2__nlist_1030+24>
       0x000000000040353d <+13>:    mov    %rbx,%rax
       0x0000000000403540 <+16>:    add    $0x8,%rsp
       0x0000000000403544 <+20>:    retq   
       0x0000000000403545 <+21>:    nopl   (%rax)
    => 0x0000000000403548 <+24>:    sub    $0x18,%r15
       0x000000000040354c <+28>:    mov    0x21780d(%rip),%rax        # 0x61ad60
       0x0000000000403553 <+35>:    cmp    (%rax),%r15
       0x0000000000403556 <+38>:    jb     0x403577 <camlTimealloc2__nlist_1030+71>
       0x0000000000403558 <+40>:    lea    0x8(%r15),%rdi
       0x000000000040355c <+44>:    movq   $0x800,-0x8(%rdi)
       0x0000000000403564 <+52>:    mov    %rsi,(%rdi)
       0x0000000000403567 <+55>:    mov    %rbx,0x8(%rdi)
       0x000000000040356b <+59>:    mov    %rsi,%rax
       0x000000000040356e <+62>:    add    $0xfffffffffffffffe,%rax
       0x0000000000403572 <+66>:    mov    %rdi,%rbx
       0x0000000000403575 <+69>:    jmp    0x403534 <camlTimealloc2__nlist_1030+4>
       0x0000000000403577 <+71>:    callq  0x411898 <caml_call_gc>
       0x000000000040357c <+76>:    jmp    0x403548 <camlTimealloc2__nlist_1030+24>
    End of assembler dump.

The allocation fast path is just these five instructions, rather than
the nine used by SBCL:

    => 0x0000000000403548 <+24>:    sub    $0x18,%r15
       0x000000000040354c <+28>:    mov    0x21780d(%rip),%rax        # 0x61ad60
       0x0000000000403553 <+35>:    cmp    (%rax),%r15
       0x0000000000403556 <+38>:    jb     0x403577 <camlTimealloc2__nlist_1030+71>
       0x0000000000403558 <+40>:    lea    0x8(%r15),%rdi

Here our allocation pointer moves *down* and is kept in %r15.

Ooh, I just learned that with `ocamlopt -S` I can coax the raw
assembly out of ocamlopt, so here’s the relevant part of the function:

    .L102:  subq    $24, %r15
            movq    caml_young_limit@GOTPCREL(%rip), %rax
            cmpq    (%rax), %r15
            jb      .L103
            leaq    8(%r15), %rdi

That’s the chunk I quoted twice above.  Then the new cons node gets
initialized:

            movq    $2048, -8(%rdi)
            movq    %rsi, (%rdi)
            movq    %rbx, 8(%rdi)

Then it builds up the arguments for the tail call:

            movq    %rsi, %rax
            addq    $-2, %rax     # n - 1, in OCaml's weird tagged integer representation
            movq    %rdi, %rbx
            jmp     .L101

Finally, this is the “allocation trampoline”, six instructions in the
SBCL code:

    .L103:  call    caml_call_gc@PLT
    .L104:  jmp     .L102

In this case we just restart the allocation instead of asking the GC
to do it for us.

C pointer-bump allocation
-------------------------

I tried the OCaml approach in C, getting even higher speeds:

    kmregion *p = km_start(km_libc_disc, &err);
    if (!p) abort();
    struct n { int i; struct n *next; } *list = NULL;

    for (size_t j = 0; j < 5000; j++) {
      struct n *q = km_new(p, sizeof(*q));
      q->i = j;
      q->next = list;
      list = q;
    }

    ...
    km_end(p);

Here `km_new` is defined in the header file as follows:

    static inline void *
    km_new(kmregion *r, size_t n)
    {
      n = (n + alignof(void*) - 1) & ~(alignof(void*) - 1);
      size_t p = r->n - n;
      if (p <= r->n) {
        r->n = p;
        return r->bp + p;
      }

      return km_slow_path_allocate(r, n);
    }

This manages to allocate 520 million list nodes per second, 1.94
nanoseconds per allocation (and initialization), 40% faster than OCaml
despite not allocating a register for the allocation pointer.
`km_slow_path_allocate`, which is separately compiled and thus not
inlineable, invokes malloc through a function pointer from
`km_libc_disc`, 32 kibibytes at a time, plus occasionally also
invoking it to store the array of block pointers, and also invoking
`longjmp` in case of allocation failure.  Since all this happens one
out of every 2048 allocations, the performance cost is normally
insignificant:

    void *
    km_slow_path_allocate(kmregion *p, size_t n)
    {
      if (n > block_size / 4) return km_add_block(p, n);
      void *block = km_add_block(p, block_size);
      if (!block) return NULL;
      p->bp = block;
      p->n = block_size;
      return km_new(p, n);
    }

GCC does a better job than I would have done at optimizing the inner
loop, scrambling it into a strange order that saves an unconditional
jump per iteration:

            call km_start
            testq %rax, %rax
            movq %rax, %rbp              # %rbp = kmregion pointer
            je .L21
            xorl %ebx, %ebx              # j = 0
            xorl %r13d, %r13d            # list = NULL
            jmp .L6
    .L23:
            movq 8(%rbp), %rax           # load base pointer
            movq %rdx, 0(%rbp)           # store allocation counter
            addq %rdx, %rax              # offset base pointer with allocation
    .L8:                                 #     pointer to get address
            movl %ebx, (%rax)            # store j in list node
            addq $1, %rbx                # increment j
            movq %r13, 8(%rax)           # store list pointer into list node
            cmpq $5000, %rbx             # loop counter termination test
            je .L22
            movq %rax, %r13              # point list pointer at new list node
    .L6:
            movq 0(%rbp), %rcx           # load allocation counter
            leaq -16(%rcx), %rdx         # decrement it
            cmpq %rdx, %rcx              # check against its old value
            jae .L23                     # if it decreased it didn’t underflow

            movl $16, %esi
            movq %rbp, %rdi
            call km_slow_path_allocate
            jmp .L8
    .L22:

The fast-path allocation here is 7 instructions, split between movq
leaq cmpq jae at the end of the loop, then movq movq addq at the
beginning.  The slow-path call is glued onto the end of the loop, thus
saving the unconditional jump, but maybe welding it onto the bottom of
the function as OCaml and SBCL do would be a better idea.

Failover in overflow cases
--------------------------

A number of late-bound languages (Smalltalk and Lisps, mostly, but
also recent versions of Python, as well as Ruby, naturally) handle
fixnum overflow transparently by failing over to bignum arithmetic,
which surely causes difficulty with both type inference and
performance predictability, but is sometimes worth the pain and
suffering.

Above we saw OCaml compile `(n-1)` to 

            addq    $-2, %rax

and we saw GCC compile the addition of two Fibonacci return values to

      40058b:       01 c5                   add    %eax,%ebp            # add return value to accumulator

You could imagine following such a single-instruction operation being
followed by a conditional jump on overflow to a fixup trampoline
welded onto the end of the function:

            addq    $-2, %rax
            jo      .tramp304
            ...
    .tramp304:
            movq    $-2, %rdi
            movq    %rax, %rsi
            call    addition_overflow_handler
            jmp     .resume305

The COMFY-6502 “win/lose” mechanism is tempting here: we might be
tempted to just treat such overflow traps as “lose continuations”, but
the fact is that we need to weld on a separate trampoline for each of
them — they’re more like conditionals.  But they may never rejoin the
original fast-path control flow: it may be convenient to preserve the
property that all the integer values on the fast path are fixnums, so
that we don’t have to test their tags repeatedly.  So the simple COMFY
one-entry two-exits approach may not work as well as one might hope.

Alternatively, it may be perfectly adequate to just do the type test
every time, though not as inefficiently as Ur-Scheme does; this might
result in compiling `n - 1` to fast-path code like this:

            test    $1, %rax    # use the low bit as the type tag
            jz      .tramp303   # using 1 as the fixnum type tag, like OCaml
            addq    $-2, %rax
            jo      .tramp304
    .resume305:
            ...

Note that the trampoline at `.tramp303` isn’t limited to doing bignum
arithmetic; it can do a fully general object-oriented dispatch, use an
inline cache (polymorphic or not), and so on.  This is probably not
going to make your dynamic object-oriented system run as fast as C or
OCaml — maybe 30% as fast — but it’ll probably be substantially faster
than SBCL.

Generic arithmetic with an open-coded fixnum path, and other fast paths
-----------------------------------------------------------------------

If you want to abjure overflow-to-bignum but still support generic
arithmetic operations, just open-coding the common fixnum fast path,
it’s a little more lightweight:

            test    $1, %rax    # use the low bit as the type tag
            jz      .tramp303   # using 1 as the fixnum type tag, like OCaml
            addq    $-2, %rax
    .resume305:
            ...

We should expect this tag test and conditional bailout to be similar
in cost to the allocation guard instructions in my allocation loop
above:

            cmpq %rdx, %rcx              # check against its old value
            jae .L23                     # if it decreased it didn’t underflow

We have an upper bound on that cost: the whole loop runs in 1.94 ns
per iteration.  And valgrind says this code runs about 6.7 billion
instructions when running 100'000 `km_region`s of 5000 nodes each:
about 13.4 instructions per loop iteration, so it's probably on the
order of 300 ps per such test.

There are probably a couple dozen methods or operators to which you’d
want to give an open-coded primitive path like this for normal code;
depending on the language, perhaps integer arithmetic (+ - × ÷ // % ==
< > <= >= == 1+ 1-), bit operations (^ & | << >> >>> &^),
floating-point arithmetic, if-then-else, range iteration, container
iteration, array indexing, car/cdr, some kind of polymorphic list
append, identity tests, field lookup, field access, string
concatenation, substring testing, string element access, pointer
arithmetic, pointer dereference, string length, array length, set
arithmetic (union, intersection, subtraction, elementwise
construction, membership testing), dictionary access (membership
testing, insertion, lookup, deletion), and pattern matching.  As I
count, that’s a bit over 50 operations (see file
`veskeno-exploration.md`), but probably any given language would get
most of the benefit from only the few of them that are most used in
that language.  (I think Ur-Scheme, for example, implements about 13
of them at all.)

The logic of this is that, even if, say, you have printf-style string
formatting built into your language as a built-in operator that’s also
used in other contexts for a fundamental arithmetic operation, as
Python does, actually executing that operation involves copying enough
characters around that doing a full method dispatch to get there is
only a little extra work.  So it might take, say, 100 ns to format the
string† and an extra 3 ns to do the full method dispatch, so avoiding
the dispatch might speed up your program by 3%.  But if you can cut
that 3 ns down to 0.3 ns when the operation is, say, an 0.3-ns
multiplication instruction, you’ve sped up your program by 5×, a 400%
increase.  So if you have to choose, it’s usually better to speed up
some programs by 400% than others by 3%.  It means that at least your
language can do *some* things fast, while the other choice is for it
to do everything slowly.

This approach will give better performance if you design the language
to avoid the case where multiple different types implement *the same
operation* in ways that are all important to optimize in this way; the
usual suspects are integer and floating-point arithmetic, both of
which can be very fast on modern machines, but which commonly use the
same operators, such as `+` and `*`.  OCaml instead uses `+.` and `*.`
for floating-point arithmetic, but other alternatives include doing
your floating-point math with numpy-like vector types which can
amortize the dispatch overhead over a number of array members.

-----

†I just did a test with snprintf in C and it took more like 400 ns per
iteration to fill a 100-byte buffer!  A small format string change cut
it to 150 ns, producing different output.