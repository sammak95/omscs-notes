# Synchronization

## 1. Lesson Introduction

This lesson will examine how to provide efficient **synchronization** among cores and threads that are working on the ***same*** program.

## 2-3. Synchronization Example

### 2. Example

<center>
<img src="./assets/20-001.png" width="650">
</center>

(***N.B.*** The figure shown above erroneously uses `R` rather than `R1` in the relevant "downstream" instructions. This error is ***not*** subsequently repeated in the following text, which uses `R1` exclusively/correctly accordingly.)

Consider an example (as in the figure shown above) to demonstrate why **synchronization** is necessary.

The system in question is characterized as follows:
  * Two threads count the occurrences of different letters in a document
  * Thread `A` counts the first half of the document
  * Thread `B` counts the second half of the document
  * Lastly, the counting of the threads is combined in the end

The corresponding program code is as follows:

(thread `A`)
```mips
LW  L, 0(R1)     # load a letter into `L`
LW  R1, Count[L] # load count for letter `L` into memory
ADD R1, R1, 1    # increment the count
SW  R1, Count[L] # load the new count back into memory
```

(thread `B` - same work as thread `A`)
```mips
LW  L, 0(R1)     # load a letter into `L` -- N.B. `R1` is distinct from that of thread A
LW  R1, Count[L] # load count for letter `L` into memory
ADD R1, R1, 1    # increment the count
SW  R1, Count[L] # load the new count back into memory
```

As long as the letters processed are ***different*** between the threads, then this program will work normally.

However, if both threads encounter the ***same*** letter (e.g., `'A'`), then both threads attempt to load the ***same*** counter value (i.e., `Count[L]`, having current value `15`).

On increment (i.e., `ADD R1, R1, 1`), both threads update the value (i.e., `16`) and store it into memory (i.e., `SW R1, Count[L]`). However, since there were ***two*** occurrences of the letter (i.e., `'A'`), this count is incorrect (i.e., `16` rather than `17` is stored in memory).

Therefore, to ensure ***correct*** program behavior, the count-incrementing operations must be performed ***sequentially*** across the two threads when handling the ***same*** data.
  * For example, if thread `A` increments first, then cache coherence ensures that the value `16` is read by thread `B`.
  * Thread `B` then subsequently increments the value to `17`.

***N.B.*** The ordering of these two operations among the threads is not significant (i.e., the equivalent holds if Thread `B` had written first instead).

These thread-coordinated operations are called **atomic (or critical) sections**. Synchronization is therefore necessary to perform such operations accordingly (i.e., additional code which ensures such thread-wise operation accordingly).

### 3. Lock

<center>
<img src="./assets/20-002.png" width="650">
</center>

Continuing from the previous example (cf. Section 2), the type of synchronization used for atomic sections is called **mutual exclusion** (or **lock**). Such a lock is used to "flank" the atomic section in order to coordinate accordingly among the threads.

<center>
<img src="./assets/20-003.png" width="650">
</center>

To perform this locking, an explicit mechanism of `lock` and `unlock` is used (as in the figure shown above).

The lock `CountLock[L]` has a status of open/closed at any given time.
  * When the lock `CountLock[L]` is ***open***, then the atomic section can be entered.
  * Otherwise, when the lock `CountLock[L]` is ***closed***, then **spinning** will occur by the thread, until the lock is once again opened.

On ***acquisition*** of the lock (and consequently closing the lock to other threads), the thread enters the atomic section and performs its operations. On exit of the atomic section, the lock becomes unlocked and subsequently available to another thread.

By having the lock present in this manner, this enforces mutual exclusion of the atomic-section code. However, this does not otherwise impose ***order*** among execution by the respective threads (it simply prevent ***simultaneous*** execution at any given time).

## 4. Lock Variable Quiz and Answers

<center>
<img src="./assets/20-005A.png" width="650">
</center>

Consider the following code fragment denoting an atomic section:

```c
lock(CountLock[L]);
count[L]++;
unlock(CountLock[L]);
```

How is `CountLock[L]` described in this context? (Select the correct option.)
  * Just another location in shared memory
    * `CORRECT`
  * A location in a special synchronization memory
    * `INCORRECT`
  * A special variable without a memory address
    * `INCORRECT`

***Explanation***:

A lock is essentially just another "variable" like any other, having a memory address, which in turn can be loaded, modified, etc.

This lesson will subsequently explore the nature of these lock variables and associated functions `lock()` and `unlock()` accordingly.

## 5-7. Lock Synchronization

### 5. Introduction

<center>
<img src="./assets/20-006.png" width="650">
</center>

To further examine lock-based synchronization, consider the following function definitions:

```cpp
typedef int mutex_type;

void lock_init(mutex_type &lock_var) {
  lock_var = 0;
}

void lock(mutex_type &lock_var) {
  while (lock_var == 1);
  lock_var = 1;
}

void unlock(mutex_type &lock_var) {
  lock_var = 0;
}
```

For simplicity, an integer is used here to represent "*a location in (shared) memory*" (cf. Section 4).

The function `lock_init()` initializes the lock variable `lock_var` to `0` (unlocked).

The function `lock()` "spins" on value `1` (locked) via `while` loop until the `lock_var` is set to `0`. On exit of the `while` loop, `lock()` sets `lock_var` to `1` (i.e., the lock is acquired, for subsequent entry into the critical section).

The function `unlock()` sets the lock `lock_var` to `0` on exit from the critical section, thereby opening/freeing the lock for subsequent use.
  * Coherence ensures that the other thread(s) waiting on `lock_var` within function `lock()` at this point observe this update to value `lock_var` accordingly (i.e., for subsequent lock acquisition)

However, note that the function `lock()` does not work in practice as implemented here.
  * Suppose there are two threads (one purple, one green), which both initially encounter the `while` loop with `lock_var` having value `0` and subsequently *both* acquire the lock via setting of `lock_var` to `1`.
  * Now, *both* threads are simultaneously accessing the critical section.

Therefore, in order to ***correctly*** implement the function `lock()`, both the ***checking*** and ***setting*** of the lock value `lock_var` must be ***atomic operations*** (i.e., performed in its own critical section accordingly).

This gives rise to an apparent ***paradox***: A critical section is needed in order to implement a critical-section-based lock.

### 6. Implementing `lock()`

<center>
<img src="./assets/20-007.png" width="650">
</center>

Per the apparent "paradox" identified in the previous section (cf. Section 5), some sort of "magic lock" modification is necessary, as follows:

```cpp
void lock(mutex_type &lock_var) {
Wait:
  lock_magic();
  if (lock_var == 0) {
    lock_var = 1;
    unlock_magic();
    goto Exit;
  }
  unlock_magic();
  goto Wait;

Exit:
}
```

Of course, there is no such "magic." Instead, the correspondingly available ***resolution measures*** for this issue are as follows:
  * Lamport's **bakery algorithm** (or another comparable algorithm) which is able to use normal load/store instructions in this manner
    * This approach is fairly ***expensive*** and complicated to implement (i.e., tens of instructions), however.
  * Use special ***atomic*** read and write instructions

### 7. Atomic Instruction Quiz and Answers

<center>
<img src="./assets/20-009A.png" width="650">
</center>

To implement atomic instructions for locks relatively easily (i.e., without otherwise resorting to complicated algorithms), which of the following properties is necessary? (Select the correct option.)
  * A load instruction (read memory)
    * `INCORRECT`
  * A store instruction (write memory)
    * `INCORRECT`
  * An instruction that *both* reads *and* writes memory
      * `CORRECT`
  * An instruction that does not access memory at all
    * `INCORRECT`

***Explanation***:

Recall (cf. Section 6) that an instruction of the following general form must be performed:

```c
if (lock_var == 0) {
  lock_var = 1;
}
```

This entails both a read (i.e., `lock_var == 0`) and a write (i.e., `lock_var = 1`).

## 8-13. Atomic Instructions

### 8. Introduction: Part 1

<center>
<img src="./assets/20-010.png" width="650">
</center>

Recall (cf. Section 7) that in order to implement atomic instructions, *both* read *and* write operations are necessary. There are three main ***types*** of such atomic instructions accordingly.

The first such atomic instruction is an **atomic exchange** instruction, which perform the following transformation:

(*instruction*)
```mips
EXCH R1, 78(R2)
```

(*transformation*)
```c
R1 = 1;
while (R1 == 1)
  EXCH R1, lock_var;
```

The instruction `EXCH` resembles a load (`LW`) or store (`SW`) instruction, however, it essentially performs both simultaneously.

In the transformed version, the value stored in `R1` is managed (i.e., exchanged) via `lock_var`. In this manner, looping persists until `R1` succeeds in obtaining the value `0`, which in turn atomically sets the variable `lock_var` to `1` accordingly (i.e., thereby acquiring the lock and precluding other threads from doing so at this point as well).

A key ***drawback*** of this atomic exchange instruction is that it writes ***persistently*** to the memory location, even while the lock is ***busy*** (i.e., locked by another thread).

The second type of atomic instruction is a ***family*** of instructions which are generally classified as **test-and-write**.
  * First, the location is ***tested***, and then if it satisfies some ***conditions***, then (and only then) ***writing*** occurs.

For example, consider a representative instruction `TSTSW` (i.e., test-and-store word), as follows:

(*instruction*)
```mips
TSTSW R1, 78(R2)
```

(*transformation*)
```c
if (Mem[78 + R2] == 0) {
  Mem[78 + R2] = R1;
  R1 = 1;
} else {
  R1 = 0;
}
```

The idea here is to test whether the lock is free (i.e., via condition `Mem[78 + R2] == 0`), and if so, then write a value `1` to the lock; otherwise, if not, then simply proceed as usual (i.e., without otherwise attempting to access the lock).

### 9. Test-and-Set Quiz and Answers

<center>
<img src="./assets/20-012A.png" width="650">
</center>

Consider a test-and-set atomic instruction of the general form `TSET R1, Addr`, defined equivalently as follows:

```c
if (Mem[Addr] == 0) {
  Mem[Addr] = R1;
  R1 = 1;
} else {
  R1 = 0;
}
```

***N.B.*** `Addr` is a memory address (typically in the form of a register and its corresponding offset).

For the corresponding implementation for function `lock()`:

```cpp
lock(mutex_type &lock_var) {
  R1 = 0;
  while (R1 == 0) {
    // TODO: Specify instruction here
  }
}
```

How can this atomic instruction be used here accordingly?

***Answer and Explanation***:

```cpp
lock(mutex_type &lock_var) {
  R1 = 0;
  while (R1 == 0) {
    TSET R1, lock_var; // Answer
  }
}
```

Here, `TSET R1, lock_var` checks the status of `lock_var`, and will only perform action if the value of `lock_var` is currently `0`, otherwise it will simply "fall through" on read of value `1`.

### 10. Atomic Instructions Part 2

<center>
<img src="./assets/20-013.png" width="650">
</center>

Returning to the topic of atomic instructions (cf. Section 8), with respect to test-and-store atomic instruction, a representative instruction `TSTSW` (i.e., test-and-store word) can be implemented as follows:

(*instruction*)
```mips
TSTSW R1, 78(R2)
```

(*transformation*)
```c
do {
  R1 = 1;
  TSTSW R1, lock_var;
} while (R1 == 0)
```

***N.B.*** This is an alternate implementation to that shown in Section 8.

Given instruction `TESTSW`, as long as the lock is occupied, then as long as the store "sub-operation" is only ***reading*** the lock variable, then it is not otherwise ***writing*** to it.
  * With respect to coherence (cf. Lesson 19), this is ***desirable***, as this prevents superfluous broadcasts of write invalidations across the respective threads' copies of the data. Instead, the "waiting" threads simply "share" the lock variable, iterating on their respective cached copies of the data at this point.
  * Subsequently on freeing of the lock, these shared copies are consequently ***invalidated*** via write to the lock variable using corresponding `unlock()`, at which point the update is broadcasted to the other threads.
  * Therefore, cross-thread communication with respect to the lock occurs only on acquisition.

This test-and-write approach solves the problem of continuously writing to the lock variable, however, it still involves the otherwise "strange" instruction `TSTSW` (and similar), i.e., an instruction which is neither a "pure load" nor a "pure store."
  * From a processor design standpoint, it would be more ideal to perform an operation that is more similar to correspondingly "pure" load and store operations, but providing the corresponding atomic functionality.

To address this particular issue, there is a third type of atomic instruction, comprising a pair called **load linked / store conditional (LL/SC)** (as discussed in the next section).

### 11. Load Linked (LL) / Store Conditional (SC)

<center>
<img src="./assets/20-014.png" width="650">
</center>

Atomic read and atomic write operations in the ***same*** instruction (even if the atomic write only occurs on detection of appropriate condition via atomic read) is very ***bad*** for pipelining insofar as processor design is concerned.

Consider a classical five-stage pipeline (as in the figure shown above).
  * A load instruction is fetched (F), decoded/read (D/R), has its address computed (A), is accessed via this computed address from memory (M), and then finally written to the register (W).
  * Given an atomic read/write operation, on reaching stage M, it cannot perform all necessary tasks in only ***one*** memory access operation (i.e., reading or writing are mutually exclusive in any given cycle at this point in the pipeline, without otherwise complicating the stage M). Therefore, just for this special case of an atomic read/write operation, the stage M would have to be expanded to a two- or three-stage (or more) memory stage (and correspondingly increasing the size of the pipeline accordingly) in order to perform appropriate checks immediately prior to writing to memory.
    * ***N.B.*** Such an expanded pipeline would impact ***all*** other instructions, not just these particular atomic read/write operations!

Consequently, the paired atomic instructions **load linked (LL)** and **store conditional (SC)** resolve this issue accordingly by splitting these atomic operations into two distinct instructions.
  * **Load linked (LL)** implements the "read" atomic operation
    * Load linked (LL) behaves as a "normal load" instruction (i.e., `LW`), simply reading from a memory location and placing the corresponding value into the target register
    * However, load linked (LL) additionally ***saves*** the address from which it loaded into a special **link register**
  * **Store conditional (SC)** implements the "write" atomic operation
    * Store conditional (SC) first checks if the address that it computes is present in the link register
      * If the address ***is*** present in the link register, then store conditional (SC) performs a "normal store" instruction (i.e., `SW`) to the memory location in question, and consequently returns the value `1` in its register
      * Conversely, if the address is ***not*** present in the link register, then store conditional (SC) simply returns the value `0` in its register (without otherwise storing a value)

Collectively, these instruction pairs effectively form a "single/composite" atomic operation via the link register.

### 12. How is Load Linked (LL) / Store Conditional (SC) Atomic?

<center>
<img src="./assets/20-015.png" width="650">
</center>

So, then, how exactly do load linked (LL) and store conditional (SC) behave in an ***atomic*** manner, such that intermediate writes from other processors/cores do not otherwise interfere with this paired operation?

To achieve this atomicity, the instruction pairs are used as follows:

```mips
LL R1, lock_var
SC R2, lock_var
```

If load linked (LL) detects the variable `lock_var` in the appropriate state (i.e., `1`), then the subsequent store conditional (SC) should only succeed if the lock is still available at the point of execution, otherwise, if the lock has already been written to, then store conditional (SC) should fail.

In order to ensure this desired behavior, the key to accomplish this is to ***snoop*** the writes to `lock_var` and to set the link register to value `0` if the lock is already acquired accordingly. Consequently, matching of the addresses by store conditional (i.e., `R2` and `lock_var`) will fail.
  * Therefore, there is a reliance on ***cache coherence*** (cf. Lesson 19) to accomplish synchronization in this manner.

Note that both load linked (LL) and store conditional (SC) are both individually atomic operations. Therefore, it is not necessary to use a "sub-lock" to further ensure atomicity with respect to either of these instructions.

Correspondingly, observe the following transformation:

(*instruction*)
```c
lock(...);
var++;
unlock(...);
```

(*transformation*)
```c
Try:
  LL R1, var;
  R1++;
  SC R1, var;
  if (R1 == 0)
    goto Try;
```

If multiple threads perform these operations, they will simply "compete" for `R1` in this manner, with the first to reach `R1` "succeeding," and the other threads consequently "failing" the condition `R1 == 0`.
  * ***N.B.*** This specific example increments `var` atomically, however, the same general principle/construct applies for other critical-section instructions performed in this manner as well.

Therefore, load linked / store conditional give rise to relatively ***simple/direct*** implementation of a critical section with corresponding atomic operations accordingly, otherwise ***obviating*** the need for locks in the transformed implementation (but rather using these atomic instructions load linked and stored conditional ***directly*** on the variable in question itself [i.e., `var`]).
  * ***N.B.*** For more complicated critical sections, such as those accessing multiple variables simultaneously, this does not work as well, however.

### 13. Load Linked (LC) / Store Conditional (SC) Quiz and Answers

<center>
<img src="./assets/20-017A.png" width="650">
</center>

Given the following function `lock()`:

```cpp
void lock(mutex_type &lock_var) {
  try_lock:
    MOV R1, 1
    LL  R2, lock_var
    SC  R1, lock_var
    // TODO: Select instructions
}
```

The intended behavior of function `lock()` is such that on exit of the function, the lock (i.e., `lock_var`) has been acquired *exclusively* (i.e., setting `R1` to value `1` accordingly).

How can load linked (LL) / store conditional (SC) be used to implement this function as follows (where `LINK` is the link register, as applicable)? (Select the applicable set of instructions.)

(*Instructions 1*)
```mips
BEQZ R1, try_lock
```
  * `INCORRECT`

(*Instructions 2*)
```mips
BEQZ R2, try_lock
```
  * `INCORRECT`

(*Instructions 3*)
```mips
BNEZ R2, try_lock
BEQZ LINK, try_lock
```
  * `INCORRECT`

(*Instructions 4*)
```mips
BNEZ R2, try_lock
BEQZ R1, try_lock
```
  * `CORRECT`

(*Instructions 5*)
```mips
BNEZ R2, try_lock
BNEZ LINK, try_lock
```
  * `INCORRECT`

***Explanation***:

All of these prospective instructions sets repeatedly attempt to acquire the lock `lock_var` via `try_lock` via appropriately provided conditions.

In order to select the *correct* conditions to accomplish this, two conditions must be checked, as follows:
  * 1 - Was a free lock observed with respect to `LL R2, lock_var`? If so, then rather than jumping back to `try_lock`, instead proceed out of the function `lock()` accordingly.
  * 2 - Otherwise, if a free lock is *not* observed, then jump back to `try_lock`.

Instruction `BNEZ R2, try_lock` satisfies both conditions.
  * A *non-zero* value in `R2` indicates that the lock is busy, necessitating going back to `try_lock`.

Additionally, it must be determined whether `R1` has been stored successfully, prior to another thread already accomplishing this. `R1` is stored successfully in this manner if `R1` contains the value `1` subsequently to (given) instruction `SC R1, lock_var`.

Instruction `BEQZ R1, try_lock` satisfies this determination accordingly.
  * If `R1` fails to store `0` via upstream instruction `SC`, then the function must jump back to `try_lock` accordingly.

Therefore, the correct set of instructions is Instructions 4. In summary, in the implementation of `lock()` using this pair of instructions:
  * If the lock is observed as free (via `BNEZ R2, try_lock` with `R2` having value `0`) *and* managing to store it at this point (via `BEQZ R1, try_lock` with `R1` having value non-zero), then the lock is acquired and the function `lock()` consequently exits accordingly.
  * Otherwise, if the lock is observed as busy/acquired (via `BNEZ R2, try_lock` with `R2` having value non-zero), then the function `lock()` jumps back to `try_lock` for subsequent attempt to acquire the lock.
  * Otherwise, if the lock is observed as free (via `BNEZ R2, try_lock` with `R2` having value `0`) but having been stored by another thread already at this point (via `BEQZ R1, try_lock` with `R1` having value `0`), then the function `lock()` jumps back to `try_lock` for subsequent attempt to acquire the lock.

***N.B.*** Instructions 3 and 5 check the link register (`LINK`) directly, however, note that this is a "hidden/abstracted" register, which generally is not "checked directly" in this manner. The corresponding access is performed ***implicitly*** by the atomic operations linked load (LL) / store conditional (SC) accordingly.

## 14-17. Locks and Performance

### 14. Introduction

<center>
<img src="./assets/20-018.png" width="650">
</center>

Consider now how different lock implementations interact with the coherence protocols (cf. Lesson 19), and what the corresponding implications are with respect to ***performance***.

Consider a system of three cores numbered `0` through `2`, as in the figure shown above.

The following sequence is performed involving atomic operation `EXCH`:

| Sequence | Cache `0` | Cache `1` | Cache `2` |
|:--:|:--:|:--:|:--:|
| `S1` | `EXCH R1, lock_var`| (N/A) | (N/A) |
| `S2` | (N/A) | `EXCH R1, lock_var`| (N/A) |
| `S3` | (N/A) | (N/A) | `EXCH R1, lock_var`|
| ⋮ | ⋮ | ⋮ | ⋮ |
| `SN` | `lock_var = 0` | (N/A) | (N/A) |

In sequence `S1`:
  * `lock_var` transitions to the modified (M) state with respect to cache `0`
  * `lock_var` is absent (i.e., invalid) in the other two cores

Eventually, cache `0` (which acquires the lock in sequence `S1`) will unlock by setting `lock_var = 0` (i.e., in downstream sequence `SN`). However, several events transpire in the intervening sequences prior to this.

In sequence `S2`:
  * Since `R1` has value `1` immediately prior to this (via cache `0`'s prior acquisition in sequence `S1`), on attempted acquisition by cache `1`, there is a corresponding transfer of the lock to cache `1`.
  * Cache `0` writes to `lock_var` and broadcasts this update, and consequently the modified (M) state is transferred from cache `0` to cache `1` accordingly.

Similarly, in sequence `S3`:
  * Since `R1` has value `1` immediately prior to this (via cache `1`'s prior acquisition in sequence `S2`), on attempted acquisition by cache `2`, there is a corresponding transfer of the lock to cache `2`.
  * Cache `1` writes to `lock_var` and broadcasts this update, and consequently the modified (M) state is transferred from cache `1` to cache `2` accordingly.

Subsequently to sequence `S3`, as long as the lock is busy in any given sequence, the other cores will ***spin*** accordingly. Consequently, the cache block will move among cores many times before the lock is finally freed/unlocked (i.e., via `lock_var = 0` in downstream sequence `SN`). This additionally incurs communication on the shared bus, with corresponding power consumption accordingly, since none of this activity succeeds in reacquiring the lock (i.e., until `lock_var` resets to `0`).

If at some later point cache `1` is the core which acquires the lock most recently (i.e., immediately following sequence `SN`, during which cache `0` is in the modified [M] state), the corresponding write will consequently transfer the modified (M) state back to cache `1` accordingly. Downstream of this, similar superfluous spinning and power consumption will commence as before.

Therefore, this is a very ***heavily power-consumptive*** process, as each such movement of the cache block requires a lot of energy per transfer operation. Furthermore, the interconnection among the cores (e.g., shared bus) experiences a high volume of traffic, adding corresponding "throttling" to the system accordingly (e.g., the one "active" core at any given time, if experiencing cache misses, will also handle these more slowly as a result, thereby effectively reducing the ***useful work*** performed by the system as a whole accordingly).

### 15. Test-and-Atomic-Op Lock

<center>
<img src="./assets/20-019.png" width="650">
</center>

Recall (cf. Section 14) that performing atomic read and atomic write operations on the lock repeatedly (even while waiting for the lock) is inefficient with respect to power consumption, and correspondingly can even have a deleterious effect on the operation of the thread currently performing useful work (i.e., inside of the critical section).

In order to reduce this inefficiency, a **test-and-atomic-op lock** approach can be used accordingly. A representative example of this can be implemented as follows:

(*original version - repeated atomic op*)
```c
R1 = 1;
while (R1 == 1)
  EXCH R1, lock_var;
```

(*improved version - test-and-atomic-op approach*)
```c
R1 = 1;
while (R1 == 1) {
  while (lock_var == 1); // use "normal" load operations
  EXCH R1, lock_var;
}
```

In the original version, "bare" `EXCH R1, lock_var` yields persistent invalidations (and corresponding cache misses) while waiting on the lock to be freed.

Conversely, in the improved version, "normal" load operations are used while the lock (i.e., `lock_var`) is busy. Once it is observed that the lock is free (i.e., `lock_var == 0`), then (and only then) there is a subsequent attempt to acquire the lock using `EXCH` (but ***not*** via "normal" store operation, as an atomic operation is necessary to acquire the exclusively lock at this point).

With this corresponding update, the following sequence is performed involving atomic operation `EXCH` (cf. similar sequence from previously in Section 14):

| Sequence | Cache `0` | Cache `1` | Cache `2` |
|:--:|:--:|:--:|:--:|
| `S1` | `EXCH R1, lock_var`| (N/A) | (N/A) |
| `S2` | (N/A) | `EXCH R1, lock_var`| (N/A) |
| `S3` | (N/A) | (N/A) | `EXCH R1, lock_var`|
| ⋮ | ⋮ | ⋮ | ⋮ |

In sequence `S1`, cache `0` acquires the lock in the modified (M) state (the other two caches are in the invalid [I] state at this point).

In sequence `S2`, cache `1` attempts to acquire the lock and transfers the cache block to itself via the shared (S) state, while cache `0` transitions to the owned (O) state with respect to this cache block. While cache `1` is in the shared (S) state, subsequent loads will yield cache hits, rather than cache misses, thereby obviating the need to access the bus.

Similarly, in sequence `S3`, cache `1` attempts to acquire the lock and transfers the cache block to itself via the shared (S) state. While cache `2` is in the shared (S) state, subsequent loads will similarly yield cache hits, rather than cache misses, thereby obviating the need to access the bus.

Therefore, while cache `0` owns the lock, the other two caches do not correspondingly generate superfluous bus traffic during this time; effectively, cache `0` has exclusive access over the coherence bus during this time, thereby allowing it to service cache misses much more quickly and efficiently. Furthermore, cache hits on cache `0` are also correspondingly much more efficient compared to coherence bus accesses accordingly.

The system proceeds in this manner until cache `0` ultimately releases the lock, at which point there is a consequent invalidation and subsequent read by the other two cores in the system (which now compete for the lock). However, now, these "activity bursts" are confined strictly to these (relatively brief) "lock exchange" periods only. Otherwise, during periods when the lock is "busy," the other caches simply persistently use "normal" loads in the meantime, which is correspondingly much more energy-efficient.
___
***Elaboration***
Sure! Let’s break this down into simpler terms, step by step.

#### Background:
In the previous example, when a core is trying to acquire a lock, it repeatedly performs an **atomic operation** (`EXCH`) to check and set the lock. This process is **inefficient** because it causes **constant communication** between the cores, even while the core is waiting for the lock. This leads to **wasted power** and **slower performance**.

#### The Problem with the Original Version:
In the **original version**, the core keeps doing the atomic exchange (`EXCH`) repeatedly in a loop while it waits for the lock. During this time:
- The core constantly checks the lock by performing atomic operations.
- Each time it checks, the system generates **cache misses** and **bus traffic** (communication between cores), which consumes a lot of power and slows down the system.

#### The Improved Version (Test-and-Atomic-Op):
The improved version uses a **test-and-atomic-op approach** to make things more efficient. Here's how it works:

1. **Check the lock first (normal load)**: Instead of constantly using the atomic operation, the core first checks the lock using a regular **load** operation. This is not atomic—it’s a simple read to see if the lock is free (if `lock_var == 0`).
   
   - If the lock is still held (i.e., `lock_var == 1`), the core just **waits** by repeatedly checking the lock using normal loads. This doesn’t generate as much **traffic** or **power usage** because it's just reading the value without making changes.
   
2. **Acquire the lock (atomic exchange)**: Once the core sees that the lock is free (i.e., `lock_var == 0`), it performs the **atomic exchange (`EXCH`)** to acquire the lock.

#### Why This is More Efficient:
1. **Reduced Traffic**: In the original version, each atomic exchange (`EXCH`) causes communication between the cores and the shared bus. In the improved version, the core only checks the lock using normal loads, which don’t cause communication unless the lock is available.
   
2. **Efficient Use of Caches**: While waiting for the lock, the other cores simply **read the lock’s status** using normal loads. This means they can use **cached copies** of the lock data, rather than repeatedly accessing the shared bus. This makes things faster and uses less power.

3. **Better Performance**: With the improved version, the system spends less time generating unnecessary communication between the cores. The only time the cores generate traffic is when the lock is actually acquired or released. During most of the time, the cores are just waiting quietly with minimal power usage.

#### Key Idea:
- **Before acquiring the lock**: Cores just check the lock using normal reads (no atomic operations).
- **When the lock is available**: The core uses an atomic operation (`EXCH`) to acquire it.

This reduces the **wasted power** from unnecessary atomic operations and **speed up** the system since there’s less communication happening when the cores are waiting for the lock.

#### How the System Works:
1. **Cache 0** acquires the lock and keeps it in its cache. The other caches (Cache 1 and Cache 2) are **inactive** and don’t need to communicate much with the bus while waiting.
   
2. **Cache 1** and **Cache 2** also check the lock with normal loads, and if it’s still locked, they don’t cause unnecessary traffic. When the lock becomes available, they try to acquire it using the atomic operation.

3. **Efficient Lock Release**: When Cache 0 releases the lock, the other caches will **invalidate their cached copies** and then **compete for the lock** using the atomic operation.

#### Conclusion:
The improved version reduces unnecessary **bus traffic** and **power consumption** by allowing cores to simply check the lock with normal loads while they wait, only using atomic operations when necessary. This leads to more **efficient performance** with less wasted energy.

---

### 16. Test-and-Atomic-Op Quiz and Answers

<center>
<img src="./assets/20-021A.png" width="650">
</center>

Recall (cf. Section 13) the following implementation for function `lock()`:

```cpp
void lock(mutex_type &lock_var) {
  try_lock:
    MOV  R1, 1
    LL   R2, lock_var
    SC   R1, lock_var // Instruction A
    BNEZ R2, try_lock // Instruction B
    BEQZ R1, try_lock // Instruction C
}
```

For the corresponding test-and-atomic-op operations, reorder instructions `A`, `B`, and `C` appropriately.

***Answer and Explanation***:

```cpp
void lock(mutex_type &lock_var) {
  try_lock:
    MOV  R1, 1
    LL   R2, lock_var
    BNEZ R2, try_lock // Instruction B′
    SC   R1, lock_var // Instruction A′
    BEQZ R1, try_lock // Instruction C
}
```

The idea with the test-and-atomic-op version is to simply refrain from writing until the lock is observed to be free. Therefore, prior to attempting to store (Instruction A), first the corresponding check of the lock is performed accordingly (Instruction B).
  * On loading the value into `R2` (i.e., `LL R2, lock_var`), if the value is `1`, then the lock is ***not*** free and correspondingly the function jumps back to `try_lock`. This pattern continues as long as the lock busy. Correspondingly, the operation load linked (LL) simply performs a "normal" load if the corresponding link register is not populated (and correspondingly there is no further "noisy traffic" among the caches).
  * Subsequently, when `R2` eventually loads value `0` (i.e., the lock becomes free), then (and only then) a subsequent store conditional (SC) operation is performed (i.e., `SC R1, lock_var`). Additionally, the subsequent check is performed to ensure that the lock has actually been acquired (i.e., `BEQZ R1, try_lock`).

### 17. Unlock Quiz and Answers

<center>
<img src="./assets/20-023A.png" width="650">
</center>

As a follow-up, now consider the corresponding test-and-atomic-op implementation for the function `unlock()`, as follows:

```cpp
void unlock(mutex_type &lock_var) {
  // TODO: Select appropriate instruction
}
```

Which of the following is the appropriate choice for this implementation? (Select the correct option.)
  * `SW 0, lock_var`
    * `CORRECT`
  * `LL`, check if `1`, then `SC`
    * `INCORRECT`
  * Additional atomic instructions are required
    * `INCORRECT`

***Explanation***:

```cpp
void unlock(mutex_type &lock_var) {
  SW 0, lock_var;
}
```

Unlike the function `lock()` (cf. Section 16), which required two checks (whether the lock is available, and then whether it can be acquired), the function `unlock()` is only performed ***exclusively*** by the thread possessing the lock at any given time. Therefore, to free the lock, `lock_var` is simply set to `0` accordingly, without otherwise requiring any additional atomic operations to accomplish this.

## 18-22. Barrier Synchronization

### 18. Introduction

<center>
<img src="./assets/20-024.png" width="650">
</center>

Another type of synchronization that is often needed in programs is called **barrier synchronization**. This often occurs when there is a parallel section wherein several threads perform independent actions simultaneously (as in the figure shown above), e.g., each thread adds the numbers in a designated sub-array section of a shared array.

Eventually, this separate work must be coordinated (e.g., determining the total sum for the entire array, across all of the threads). In order to do this, it is necessary to ensure that each thread has concluded performing its own individual work; this is provided by the **barrier** accordingly. Only on reaching the barrier in this manner can the final result be used (e.g., `print(sum);`).

Another possibility is that on computation, all of the threads must access the sum simultaneously (as in the figure shown above). For this purpose, the barrier also serves as a corresponding "checkpoint" in this manner, serving as a ***global wait*** which ensures that ***all*** threads have first entered the barrier prior to commencing with access of this "global" value. The barrier synchronization ensures that all threads have ***arrived*** first, prior to commencing with departure from the barrier.

A barrier ***implementation*** typically consists of two variables, as follows:
  * **counter** → counts how many threads have arrived
    * Each arriving thread increments this value accordingly on arrival to the barrier
  * **flag** → sets when the counter finally reaches the total threads count (i.e., condition `counter == N`)
    * Each arriving thread checks the flag on arrival, and sets accordingly (i.e., set the flag to "all have arrived," or otherwise spin while pending arrival of the remaining threads)

However, it is not quite so straightforward to implement this in practice; the remainder of this section will discuss matter accordingly.

### 19. Simple Barrier Implementation

<center>
<img src="./assets/20-025.png" width="650">
</center>

Consider the following relatively simple implementation of a barrier in a program:

```c
lock(counter_lock);          // begin critical section
if (count == 0) release = 0; // re(initialize) release flag to `0` (acquired)
count++;                     // count the arrivals
unlock(counter_lock);        // end critical section
if (count == total) {
  count = 0;                 // (re)initialize the counter to `0`
  release = 1;               // (re)set release flag to `1` (released)
} else {
  spin(release == 1);        // wait for `release` to attain value `1` (released)
}
```

The counter `count` is a shared variable, which is contended over by the threads (for subsequent incrementing).

On exit of the critical section, this implies that the thread in question is either the last thread, or that the last thread has arrived.
  * ***N.B.*** These two conditions are distinct. For example, in general, the currently incrementing thread might not necessarily be the last-arriving. However, by the point of checking the total immediately following exit of the critical section, the last thread has arrived.

Subsequently:
  * If `count == total` at this point, then the counter `counter` is re-initialized (i.e., reset to `count = 0`) for subsequent thread-wise access to the critical section. Furthermore, the release flag `release` is set to `1`, thereby informing the waiting threads that the barrier can now be released accordingly.
  * Otherwise, the thread waits via `spin()`, pending release by the last thread.

Furthermore, on entry into the critical section, the first-arriving thread resets the release flag `release` to `0` (i.e., acquired) to notify the other threads accordingly.

This simple implementation is ***not*** entirely correct, however. Consider the scenario of two threads synchronizing on this barrier (as in the figure shown above).
  * In this case, `total == 2` for the total number of threads in the system.
  * On first pass, the barrier works correctly, as intended.
  * However, on subsequent arrivals to the barrier, a **deadlock** condition can result (as discussed in the next section).

### 20. Simple Barrier Implementation Does *Not* Work

<center>
<img src="./assets/20-026.png" width="650">
</center>

Recall (cf. Section 19) the simple barrier implementation as follows:

```cpp
lock(counter_lock);          // begin critical section
if (count == 0) release = 0; // re(initialize) release flag to `0` (acquired)
count++;                     // count the arrivals
unlock(counter_lock);        // end critical section
if (count == total) {
  count = 0;                 // (re)initialize the counter to `0`
  release = 1;               // (re)set release flag to `1` (released)
} else {
  spin(release == 1);        // wait for `release` to attain value `1` (released)
}
```

Furthermore, consider two cores (numbered `0`/blue and `1`/magenta) running the same program simultaneously (as in the figure shown above).

On arrival at the critical section, assume that core `0` arrives first for the sake of example. Thread `0` (corresponding to core `0`) passes through the critical section, and eventually waits at `spin()`.
  * The analogous operation is `LW release` (yielding `0`) at this point. It continues to spin in this manner, checking the status of `release`.

Subsequently, on arrival at the critical section, thread `1` (corresponding to core `1`), ideally it should pass through the critical section (setting `count` to `2` in the process), and then subsequently set `count` to `0` and `release` to `1` accordingly.
  * The analogous operation is `SW release` (set to  `1`) at this point, which in turn causes thread `1` to exit from `spin()` (thereby exiting the barrier accordingly).

However, consider if while thread `0` is checking the status of `release`, when the subsequent release by thread `1` occurs, there is a delay in this checking by thread `0` (e.g., an intermediate interrupt is suspended at this point, precluding a "sufficiently fast" exit from `spin()`).
  * In this case, rather than exiting `spin()` (via corresponding detection of value `1` via `LW release`), instead thread `1` proceeds to re-enter the critical section (detecting `count == 0` at this point from its own preceding release) and subsequently resetting `release` to `0` (i.e., *not* released).
  * Consequently, on subsequent check by thread `0` in `spin()`/waiting, it no longer sees a "released" system state.

Now, both threads proceed similarly as before with their respective operations, however, this time, since the counter `count` is "inaccurate," both threads will eventually end up in the `spin()`/waiting state. This condition is called a **deadlock**.

### 21. Simple Barrier Quiz and Answers

<center>
<img src="./assets/20-028A.png" width="550">
</center>

To better understand the problem with the simple barrier implementation, consider the same code from previously (cf. Section 20), as follows:

```cpp
lock(counter_lock);          // begin critical section
if (count == 0) release = 0; // re(initialize) release flag to `0` (acquired)
count++;                     // count the arrivals
unlock(counter_lock);        // end critical section
if (count == total) {
  count = 0;                 // (re)initialize the counter to `0`
  release = 1;               // (re)set release flag to `1` (released)
} else {
  spin(release == 1);        // wait for `release` to attain value `1` (released)
}
```

Now, assume that on exit from the barrier by both threads (i.e., `0` and `1`), only thread `0` proceeds to do subsequent work, while thread `1` simply "terminates."

In this case, is the simple barrier implementation effective/correct?
  * `YES`

***Explanation***:

If the barrier is only used ***once*** (i.e., via corresponding critical-section code and `release` mechanism), then eventually when they both perform the "first passthrough" of the critical section, the system will reach a state whereby thread `0` waits on `spin()`, and then thread `1` resets count to `0` and `release` to `1`. Eventually, due to coherence, the wait will resolve, and thread `1` simply exits the barrier (and subsequently terminates), while thread `0` continues onto subsequent work.
  * Alternatively, neither may end up waiting via `spin()` but rather simply both release instead, however, the net effect is the same (i.e., thread `1` terminates, and thread `0` proceeds onto subsequent work).

Therefore, the ***incorrectness*** of the simple barrier implementation does not stem from this "first passthrough" scenario, but rather that this implementation gives rise to a barrier-based release mechanism which is ***not reusable*** (i.e., on subsequent entries by threads into the barrier).

### 22. Reusable Barrier

<center>
<img src="./assets/20-029.png" width="650">
</center>

So, then, how can such a **reusable barrier** be implemented? This can be done as follows:

```cpp
local_sense != local_sense;
lock(counter_lock);
count++;
if (count == total) {
  count = 0;
  release = local_sense;
}
unlock(counter_lock);
spin(release == local_sense);
```

The idea here is that the value for releasing the barrier (i.e., `release`) in general is ***not*** the same for all instances of the barrier itself (i.e., even in those instances for which `release` becomes `0`, all other instances release on `release` reaching `1`).

Therefore, it is not actually necessary to ever *reinitialize* `release` to `0`, but rather it is simply "flipped" at any given time. Correspondingly, each thread has `local_sense`, which at any given time indicates "which" release to detect in order to exit from waiting.

For example, consider two threads (numbered `0` and `1`), as in the figure shown above. Assume that `local_sense` is initialized to `0` (respectively in each thread, with each owning this independently as its own local variable), and then subsequently detects a value of `1` immediately prior to entry of either thread into the critical section.

As the threads pass through the critical section, one thread will eventually detect the condition `count == total` and subsequently set `release = local_sense` (i.e., `1` in this case), which is the pending value for the particular barrier in question. At this point, the other thread has exited the critical section and is waiting (i.e., via `spin(release == local_sense)`). Here, the thread waits on the release condition (i.e., `local_sense` having value `1`).

Consider the situation where the setting of `release` to `1` is not detected by the waiting thread (i.e., due to "delayed" detection prior to exiting `spin()`).
  * In this case, the other thread proceeds onto the next iteration and re-enters the critical section. However, on this re-entry, it will invert its own local version of `local_sense` back to `0` and set this accordingly.
  * Now, it will fall through the critical section and itself reach the waiting condition (i.e., `spin(release == local_sense)`), however, having set `release` to `0`, the other thread now detects this as the "release" condition (i.e., per its own `local_sense` having value `0` from previously), which in turn releases this thread for subsequent work (i.e., at this point, they are waiting on a "disparate" values of `release`).
    * ***N.B.*** At this point, "globally" `release` still has a value of `1`, which is how it was set most recently.

Generalizing this manner, the released thread will subsequently "invert" the value again, thereby releasing the currently waiting thread, and so on. Therefore, this inversion allows to reuse the barrier, without otherwise risking a deadlock condition occurring in the program during execution.

## 23. Lesson Outro

This lesson has demonstrated that synchronization operations (e.g., locks and barriers) are necessary in order to coordinate the activities among multiple threads. Furthermore, special atomic operations are necessary in order to implement locks efficiently.

The next lesson will examine how synchronization can be perturbed when a processor is capable of reordering load and store operations, and consequently how to guard against this accordingly.
