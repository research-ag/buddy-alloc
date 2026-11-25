# BuddyAlloc: implementation spec

`BuddyAlloc` (mops package `buddy-alloc`) allocates and frees variable-size
byte blocks inside a single Motoko `Region` (IC stable memory). This document
specifies the first version, which never merges freed blocks.

## 1. Overview

- **One Region for all sizes.** Motoko gives a Region physical memory in
  8 MiB steps, and Regions can never shrink or be freed. So all sizes share
  one Region, with blocks of different sizes interleaved.
- **Power-of-two size classes:** from 16 bytes to 64 KiB. A larger block is
  split in halves to serve a smaller request, as in a buddy allocator.
- **A sized interface.** The caller passes the size to every call and keeps
  it. Blocks therefore have no header and store no length; an allocated block
  holds only the caller's bytes.
- **Requests over 64 KiB** get a run of whole 64 KiB pages.
- **No merging:** freed blocks are never merged with their neighbors.

## 2. Constants

| Name | Value | Meaning |
|---|---|---|
| unit | 16 bytes | All sizes are rounded up to a multiple of it |
| class n | n = 0 … 12 | Block size `16 · 2^n`: 16 B, 32 B, … 64 KiB |
| page | 64 KiB | One Wasm page, the same size as class 12; the Region grows in pages |
| `Ptr` | `Nat32` | Byte offset in the Region ÷ 16 |
| `NIL` | `0xFFFF_FFFF` | "No block", marks the end of a free list |
| max Region size | 2^20 − 1 pages | Just under 64 GiB. Every offset then fits a `Ptr`, and no block's `Ptr` equals `NIL` |

- **Alignment:** a block of class n always starts at a byte offset that is a
  multiple of its own size (`16 · 2^n`). A page starts at a multiple of 64 KiB.
- **Class of a size:** let `u` be the size rounded up to a multiple of 16.
  The class is the smallest n with `16 · 2^n ≥ u`. So sizes 1–16 are class 0,
  17–32 class 1, 33–64 class 2, and so on.

## 3. Interface

```motoko
module BuddyAlloc {
  public type Ptr = Nat32;
  public type State;   // section 4
  public type Stats = {
    regionPages : Nat;     // the Region's size in pages
    freeBlocks : [Nat];    // per class 0..12, blocks on the free list
    liveBlocks : [Nat];    // per class 0..12, blocks currently allocated
    liveRunPages : Nat;    // pages in currently allocated runs
  };

  public func new() : State;
  public func alloc(self : State, size : Nat) : Ptr;
  public func allocChecked(self : State, size : Nat) : ?Ptr;  // null instead of a trap when the Region can't grow
  public func free(self : State, p : Ptr, size : Nat);
  public func offset(p : Ptr) : Nat64;                       // byte offset in the Region: p * 16
  public func region(self : State) : Region;                 // for Region.load*/store* at offset(p)
  public func stats(self : State) : Stats;
}
```

The first parameter is named `self` so that calls can use dot notation
(`state.alloc(size)`).

**Preconditions and behavior:**
- **`alloc` / `allocChecked`:**
  - `size ≥ 1`; `alloc(0)` traps.
  - They trap or return `null` when the Region can't grow: `Region.grow`
    failed, or the Region would pass its maximum size.
- **`free`:**
  - `size` must be exactly the size passed to `alloc` for that pointer.
  - Freeing with a wrong size, freeing twice, or freeing a pointer that was
    never allocated is undefined behavior: there is no header to check
    against. The debug check in section 7 catches it in tests.
- **Block contents:**
  - The caller reads and writes within the first `size` bytes of its block,
    at `offset(p)`.
  - A newly allocated block's contents are unspecified: it may hold old data
    or free-list pointers. Only blocks cut from freshly grown pages are zero.
    The caller must write before it reads.

## 4. State

```motoko
public type State = {
  region : Region;
  heads : [var Nat32];       // 13 free-list heads, one per class; NIL when empty
  freeBlocks : [var Nat];    // 13 counters, for stats()
  liveBlocks : [var Nat];    // 13 counters
  var liveRunPages : Nat;
};
```

- **Persistence:** the State is a stable type, kept in the actor with the
  rest of its state. Changing its fields later needs a migration.
- **Free lists:**
  - Singly linked and last-in first-out, one per class.
  - A free block's first 4 bytes hold the `Ptr` of the next free block of
    the same class (little-endian `Nat32`), or `NIL`.
  - Allocated blocks hold no list data.
- **No end offset:** the whole Region is always in use, as free blocks or
  allocated ones. It grows only when an allocation needs fresh pages, and
  then by exactly those pages. `Region.grow(region, k)` returns the previous
  size in pages, which is where the new pages start. It returns
  `0xFFFF_FFFF_FFFF_FFFF` on failure.

## 5. Algorithms

```
alloc(size):
  u = roundUp(size, 16)
  if u > 64 KiB: return allocRun(u)
  n = classOf(u)
  m = smallest class >= n whose free list is not empty
  if there is none:
    block = grow(1)                     // one fresh page
    m = 12
  else:
    block = pop(m)
  while m > n:                          // split down to class n
    m -= 1
    push(block + 16 * 2^m, m)           // upper half onto the free list of class m
  return block / 16                     // the caller gets the lower half

free(p, size):
  u = roundUp(size, 16)
  if u > 64 KiB: return freeRun(p, u)
  push(p * 16, classOf(u))              // the whole class block, as it was allocated

allocRun(u):                            // u > 64 KiB
  k = ceil(u / 64 KiB)
  return grow(k) / 16                   // always k fresh pages at the end of the Region

freeRun(p, u):
  k = ceil(u / 64 KiB)
  for i in 0 ..< k: push(p * 16 + i * 64 KiB, 12)

grow(k):                                // returns the byte offset of the first new page
  if regionPages + k > 2^20 - 1: fail
  old = Region.grow(region, k)
  if old == 0xFFFF_FFFF_FFFF_FFFF: fail
  return old * 64 KiB

push(block, n): store heads[n] in the first 4 bytes of block; heads[n] = block / 16
pop(n):         block = heads[n] * 16; heads[n] = the Ptr in the first 4 bytes of block; return block
```

"fail" means trap in `alloc` and return `null` in `allocChecked`. Update the
`stats()` counters in `alloc`, `free`, `push` and `pop`.

Notes:
- **Runs never come from the free lists.** Nothing tracks which free pages
  are next to each other, so a run always takes fresh pages. A freed run's
  pages go onto the class-12 list one by one, where they serve requests of
  64 KiB or less.
- **No trimming.** A request uses its whole class block; for example, a
  91-byte request takes a 128-byte block. Don't return the unused end of a
  block to smaller free lists. A block freed that way comes back in pieces
  that a later request of the same size can't use, and under steady
  allocate/free traffic the Region keeps growing.
- **Keep the alignment.** Merging may be added in a later version. It relies
  on every block being aligned to its own size, with a block's buddy at
  offset XOR size, which the algorithm above guarantees.

**Cost:**
- **alloc:** at most 13 free-list checks, at most 12 splits (one 4-byte store
  each) and at most one `Region.grow`.
- **free:** one 4-byte store, or k stores for a run of k pages.
- **Space:** no per-block overhead. Rounding up to the class size wastes less
  than half a block, and a run wastes less than one page.

## 6. Worked examples

They run in sequence, starting from an empty allocator; offsets are in bytes.

1. `alloc(91)`:
   - Class 3 (128 B).
   - No free blocks, so the Region grows to 1 page.
   - The page is split down, pushing 32 KiB at 32768, 16 KiB at 16384, 8 KiB
     at 8192, 4 KiB at 4096, 2 KiB at 2048, 1 KiB at 1024, 512 B at 512,
     256 B at 256 and 128 B at 128.
   - Returns offset 0.
2. `free(0, 91)`: pushes the 128-byte block at 0 onto the class-3 list.
3. `alloc(16)`:
   - Classes 0–2 are empty, so it pops the 128-byte block at 0.
   - Splitting pushes 64 B at 64, 32 B at 32 and 16 B at 16.
   - Returns offset 0.
4. `alloc(100 KiB)`:
   - A run of 2 pages: the Region grows to 3 pages.
   - Returns offset 65536.
5. `free(65536 / 16, 100 KiB)`: pushes the pages at 65536 and 131072 onto the
   class-12 list.

## 7. Invariants and debug check

After every operation:
1. Every free block of class n starts at a multiple of `16 · 2^n` and lies
   inside the Region.
2. Free blocks and allocated blocks never overlap, and together they cover
   the whole Region exactly.
   - An allocated block's extent is its whole class block, or k whole pages
     for a run.
3. The Region has at most 2^20 − 1 pages.
4. The `stats()` counters match the free lists and the allocated blocks.

Provide a debug variant, used only in tests, that tracks allocated blocks
(pointer → size) and checks these invariants. It also catches a free with a
wrong size, a double free and a free of an unknown pointer.

## 8. Tests and benchmarks

- **Worked examples:** section 6 as unit tests.
- **Randomized test:**
  - Long random sequences of `alloc` and `free`.
  - Mostly small sizes (16 to a few hundred bytes), with some sizes between
    1 KiB and 64 KiB and some over 64 KiB.
  - Check the invariants periodically.
  - Write distinct bytes into every allocation, and check them before
    freeing it.
- **Churn:**
  - Fill the allocator, then run equal numbers of allocations and frees.
  - The Region's size should level off rather than keep growing.
  - Report it against the live bytes.
- **Upgrade:** the state and the block contents survive a canister upgrade.
- **Benchmark:** instructions per `alloc` and `free`, both with the needed
  class already on its free list and when a split or a grow is needed.

## 9. Not in this version

- **Merging freed buddies.** It is planned for later, using a bitmap of one
  bit per buddy pair and doubly linked free lists, and needs no change to the
  block layout.
- **Shrinking the Region.** Regions can't shrink.
- **Thread safety.** None is needed: canister messages run one at a time.
