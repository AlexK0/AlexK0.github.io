---
title: "Crafting memory allocators"
date: 2024-02-08 12:00:00 +0100
categories: [ Blogging, Development ]
tags: [ C/C++, Development ]
image: /assets/img/posts/crafting-memory-allocators/0_title.jpg
published: true
---

It's hard to overstate the importance of memory allocators. Without them, programs simply wouldn't work.
One essential step in optimizing code is using custom memory allocators.

There are many different ones available – like

- [tcmalloc](https://github.com/google/tcmalloc),
- [tbbmalloc](https://oneapi-src.github.io/oneTBB/main/tbb_userguide/Scalable_Memory_Allocator.html),
- [jemalloc](https://github.com/jemalloc/jemalloc),
- and more.

By simply linking your program with one of these allocators, you can almost effortlessly improve performance.
These allocators provide fast allocations and deallocations, good memory locality, all of which significantly impact
program performance.
However, there's no such thing as a free lunch; sometimes, using custom allocators can lead to increased memory usage
or, worse, memory fragmentation.
Therefore, choosing a memory allocator always involves some compromise.

In this post, I'd like to discuss how to build a memory allocator.
While it will be relatively simple compared to existing solutions,
understanding how to create even such a basic allocator provides essential insight into what allocators are and why
they're necessary.

# The Simple Allocator

> All the code written below is built and tested using Clang 15 with C++17 on macOS Sonoma. \
> However, it can be easily adapted for C++11 on any C++ compiler and operating system.
{: .prompt-info }

I will call it **The Simple Allocator**. **The Simple Allocator** can be used in various ways:

- Manual memory allocations and deallocations,
- Overriding standard [malloc](https://en.cppreference.com/w/c/memory/malloc)'s behavior,
- Using in STL containers.

However, **The Simple Allocator** has limitations:

- Single-threaded,
- Allocated memory must be pre-allocated, for example, on the heap, on the stack, .data segments, .BSS segment, or
  via [mmap](https://man7.org/linux/man-pages/man2/mmap.2.html),
- Potential fragmentation with long-term use.

These limitations can be relaxed or eliminated, but it will complicate the implementation. Later, when the allocator is
ready, I'll revisit these points.

As for **The Simple Allocator** beneficial properties, I would highlight:

- Pointers aligned to 16 bytes, customizable if needed,
- Good memory locality,
- On average, it will be faster than the standard malloc implementation.

## Naive implementation

Let's start from the naive implementation.

### Allocator alignment

As mentioned earlier, the allocator returns pointers aligned to 16 bytes, which can be parameterized.
The parametrization can be done in several ways:

- As a compile-time class template parameter,
- As a runtime class constructor parameter,
- Or simply as a constexpr or `#define` constant.

Utilizing template or runtime parameters offers flexibility but complicates the implementation unnecessarily.
Thus, I prefer to define the allocator alignment as a constant.
However, it would be relatively easy to rework it into a parameter if needed.

```cpp
class SimpleAllocatorTraits {
public:
  static constexpr size_t ALIGNMENT = 16;
  static_assert(ALIGNMENT && ((ALIGNMENT - 1) & ALIGNMENT) == 0, "power of 2 is expected");
};
```

> The alignment must be a power of two.
{: .prompt-info }
> Programs on x86-64 operating systems may not work properly if the alignment is less than 16.
{: .prompt-warning }

We also need a couple of helper functions that align the sizes and the pointers.

```cpp
template<size_t N, class T>
constexpr T AlignN(T v) noexcept {
  static_assert(N && ((N - 1) & N) == 0, "power of 2 is expected");
  return static_cast<T>((static_cast<uint64_t>(v) + N - 1) & (~(N - 1)));
}

template<size_t N, class T>
constexpr std::pair<T *, T *> AlignBuffer(T *begin, T *end) noexcept {
  return {reinterpret_cast<T *>(AlignN<N>(reinterpret_cast<uintptr_t>(begin))), reinterpret_cast<T *>(AlignN<N>(reinterpret_cast<uintptr_t>(end) - (N - 1)))};
}
```

Although they mostly look like creepy bitwise magic, they perform a pretty simple task.

`AlignN<N>(integer)` returns the value rounded to the next integer aligned to `N`, for example,

| call              | returns |
|-------------------|---------|
| `AlignN<16>(7)`   | `16`    |
| `AlignN<16>(25)`  | `32`    |
| `AlignN<16>(32)`  | `32`    |
| `AlignN<16>(798)` | `800`   |

`AlignBuffer<N>(begin, end)` returns the sub-interval for the buffer with pointers aligned to `N`, for example,

| call                                          | returns        |
|-----------------------------------------------|----------------|
| `AlignBuffer<16>((void *)0x7, (void *)0x27)`  | `{0x10, 0x20}` |
| `AlignBuffer<16>((void *)0x27, (void *)0x54)` | `{0x30, 0x50}` |
| `AlignBuffer<16>((void *)0x1, (void *)0x2)`   | `{0x10, 0x00}` |

As you may note, `AlignBuffer<N>()` doesn't concern itself with the correctness of the sub-interval. That's okay; we
will check it during usage.

> In further parts of the text, when I mention that something is ***aligned***, it means that it is ***aligned***
> to `SimpleAllocatorTraits::ALIGNMENT` bytes.
{: .prompt-info }

### Allocator memory block

The next important part of the implementation is a class that represents a memory block.
Memory blocks are the main primitives used by the allocator for allocating memory.

```cpp
class MemoryBlock {
public:
  constexpr explicit MemoryBlock(size_t size) noexcept: metadata{size} {}

  constexpr size_t GetBlockSize() const noexcept { return metadata.size; }
  constexpr void SetBlockSize(size_t size) noexcept { metadata.size = size; }

  uint8_t *UserMemoryBegin() noexcept { return reinterpret_cast<uint8_t *>(this + 1); }
  uint8_t *UserMemoryEnd() noexcept { return UserMemoryBegin() + metadata.size; }

  static constexpr MemoryBlock *FromUserMemory(void *ptr) noexcept {
    return static_cast<MemoryBlock *>(ptr) - 1;
  }

private:
  struct alignas(SimpleAllocatorTraits::ALIGNMENT) {
    size_t size;
  } metadata;
};
```

Most of the methods are self-explanatory. Let's discuss how the class is going to be used and why the `metadata` field
needs the explicit `alignas` specifier.

Let me jump ahead a bit and mention that all `MemoryBlock` objects are created on aligned pointers.
The user doesn't interact with the `MemoryBlock` object directly; instead, the user receives a pointer to the allocated
memory from `MemoryBlock::UserMemoryBegin()`,
and upon deallocation, the user passes this pointer back to the allocator.
We then restore the pointer to the original `MemoryBlock` using the static method `MemoryBlock::FromUserMemory()`. Here
is a pseudocode example:

```cpp
void *malloc(size_t size) {
  MemoryBlock *memory_block = allocate_memory_block_somehow(size);
  return memory_block->UserMemoryBegin();
}

void free(void *ptr) {
  MemoryBlock *memory_block = MemoryBlock::FromUserMemory(ptr);
  deallocate_memory_block_somehow(memory_block);
}
```

![MemoryBlock object layout](/assets/img/posts/crafting-memory-allocators/1-memory-block.png)
_MemoryBlock object layout_

From all this, it follows that if we intend to return the aligned pointer to the user, the `MemoryBlock::metadata` must
also be aligned. Therefore, the explicit `alignas` specifier is required.

Note that in this implementation, we only store size in the metadata, but it may be more complicated. We can include
many other details, such as:

- a flag indicating if the memory has already been deallocated to detect double deallocations,
- a linked list node containing all allocated blocks to detect memory leaks,
- call stacks indicating where the last allocation and deallocation were performed for easier debugging of the issues,
- and so on.

### Allocator interface

The class interface is quite trivial:

```cpp
class SimpleAllocator {
public:
  SimpleAllocator() = default;
  bool Init(void *buffer, size_t buffer_size) noexcept;

  void *Allocate(size_t size) noexcept;
  void Deallocate(void *ptr) noexcept;
  void *Reallocate(void *ptr, size_t new_size) noexcept;

private:
  uint8_t *CutBuffer(size_t size) noexcept;

  uint8_t *buffer_begin_{nullptr};
  uint8_t *buffer_end_{nullptr};
  uint8_t *current_{nullptr};
};
```

### SimpleAllocator::Init

At first glance, the `Init()` method may appear as follows:

```cpp
void SimpleAllocator::Init(void *buffer, size_t buffer_size) noexcept {
  buffer_begin_ = static_cast<uint8_t *>(buffer);
  buffer_end_ = buffer_begin_ + buffer_size;
  current_ = buffer_begin_;
}
```

The issue with such an implementation is that `void *buffer` may be an unaligned pointer, and `size_t buffer_size` may
represent an unaligned size.
While we could address this concern later, it is much better to maintain the invariant that all pointers in the
allocator are aligned from the beginning.

To achieve this, we can use the `AlignBuffer<SimpleAllocatorTraits::ALIGNMENT>()` method. Here is a revised version of
the `Init()` method.

```cpp
bool SimpleAllocator::Init(void *buffer, size_t buffer_size) noexcept {
  if (buffer_begin_ || buffer_end_ || current_) { return false; }

  auto being = static_cast<uint8_t *>(buffer);
  auto end = being + buffer_size;
  auto [buffer_begin, buffer_end] = AlignBuffer<SimpleAllocatorTraits::ALIGNMENT>(being, end);
  if (buffer_begin >= buffer_end) { return false; }

  buffer_begin_ = buffer_begin;
  buffer_end_ = buffer_end;
  current_ = buffer_begin_;
  return true;
}
```

The code is pretty straightforward. We pick an aligned sub-buffer and return true in case of successful initialization,
false otherwise.

![Aligned allocator buffer](/assets/img/posts/crafting-memory-allocators/2-init-buffer.png)
_Aligned allocator buffer_

### SimpleAllocator::CutBuffer

Before delving into the details, let's take a look at another helper private method, `CutBuffer()`:

```cpp
uint8_t *SimpleAllocator::CutBuffer(size_t size) noexcept {
  if (current_ + size > buffer_end_) { return nullptr; }

  uint8_t *memory_piece = current_;
  current_ += size;
  return memory_piece;
}
```

The method cuts the requested size from the allocator buffer and returns a pointer to the beginning of the cut memory.
If there is insufficient memory available, it returns `nullptr`. Note that the method doesn't concern about alignment.

![Cut allocator buffer](/assets/img/posts/crafting-memory-allocators/3-cut-buffer.png)
_Cut allocator buffer_

### SimpleAllocator::Allocate

It's time for allocation. Here is the first and main method, `Allocate()`:

```cpp
void *SimpleAllocator::Allocate(size_t size) noexcept {
  if (!size) {
    return nullptr;
  }

  size = AlignN<SimpleAllocatorTraits::ALIGNMENT>(size);

  static_assert(alignof(MemoryBlock) % SimpleAllocatorTraits::ALIGNMENT == 0);
  if (uint8_t *memory_piece = CutBuffer(sizeof(MemoryBlock) + size)) {
    auto *memory_block = new (memory_piece) MemoryBlock{size};
    return memory_block->UserMemoryBegin();
  }
  return nullptr;
}

```

The implementation is quite simple:

1. Align the allocating size.
2. Attempt to cut out a memory segment from the allocator buffer.
3. Instantiate a memory block with metadata on the cut memory segment.
4. Return the allocated memory using `MemoryBlock::UserMemoryBegin()`.

One of the most important aspects here is the `size` aligning. We cut the memory segment with a size
of `sizeof(MemoryBlock) + size` from the aligned buffer.
Because `MemoryBlock` and `size` are aligned, their sum will also be aligned. Consequently, the `current_` pointer will
be aligned after cutting the buffer.
This ensures that all pointers in the allocator are aligned, maintaining the required invariant.

> Also, you may note that even if a user requests 1 byte, the allocator allocates 16 bytes for the user and cuts 32
> bytes from the buffer.
> This could be improved; for example, we could reserve a region in the buffer for ***small memory blocks*** with a
> fixed
> size of 16 bytes,
> which don't have any metadata at all, and use them for small allocations. With this optimization, the allocator still
> allocates 16 bytes in case the user requests 1 byte,
> but it will only cut 16 bytes from the buffer. I will not implement this optimization here because I don't want to
> overcomplicate the implementation.

> This optimization cannot be extended to keep ***very small memory blocks*** for 8 bytes and
***super very small memory blocks*** for 4 bytes because, in this case, small allocations will return unaligned
> pointers.
{: .prompt-warning }

### SimpleAllocator::Reallocate

At first glance, `Reallocate()` can be implemented like this:

```cpp
void *SimpleAllocator::Reallocate(void *ptr, size_t new_size) noexcept {
  auto *new_ptr = Allocate(new_size);
  if (ptr && new_ptr) {
    auto *memory_block = MemoryBlock::FromUserMemory(ptr);
    std::memcpy(new_ptr, ptr, std::min(new_size, memory_block->GetBlockSize()));
    Deallocate(ptr);
  }
  return new_ptr;
}
```

But this is not efficient.

Here is an example:

```cpp
void *ptr = allocator.Allocate(2);
// do something
void *new_ptr = allocator.Reallocate(ptr, 8);
```

Should the allocator do anything here? Actually, no. In this case, the `Reallocate()` method may simply return the same
pointer.
The reason is that when allocating 2 bytes, the allocator returns a 16-byte `MemoryBlock`, and reallocating to 8 bytes
actually requests a 16-byte `MemoryBlock` too,
so nothing needs to be done; just return the same pointer to the same memory.

Another interesting case is when the reallocating `MemoryBlock` is adjusted to the main buffer. In this case, we may
simply extend it.

![MemoryBlock extending](/assets/img/posts/crafting-memory-allocators/4-reallocate.png)
_MemoryBlock extending_

Here is the improved implementation:

```cpp
void *SimpleAllocator::Reallocate(void *ptr, size_t new_size) noexcept {
  if (!ptr) { return Allocate(new_size); }
  if (!new_size) { Deallocate(ptr); return nullptr; }

  new_size = AlignN<SimpleAllocatorTraits::ALIGNMENT>(new_size);

  auto *memory_block = MemoryBlock::FromUserMemory(ptr);

  // The old MemoryBlock is suitable for the new size
  if (new_size == memory_block->GetBlockSize()) { return ptr; }

  // Check if the old MemoryBlock is adjusted to the main buffer
  if (memory_block->UserMemoryEnd() == current_) {
    if (new_size < memory_block->GetBlockSize()) {
      // Return unnecessary memory to the main buffer
      current_ -= memory_block->GetBlockSize() - new_size;
      memory_block->SetBlockSize(new_size);
      return ptr;
    }
    const size_t extra_size = new_size - memory_block->GetBlockSize();
    // Cut extra memory from the main buffer and add it into the old MemoryBlock
    if (CutBuffer(extra_size)) {
      memory_block->SetBlockSize(new_size);
      return ptr;
    }
  }

  auto *new_ptr = Allocate(new_size);
  if (new_ptr) {
    std::memcpy(new_ptr, ptr, std::min(memory_block->GetBlockSize(), new_size));
    Deallocate(ptr);
  }
  return new_ptr;
}
```

Interestingly, this implementation is final and will not be changed or optimized later.

### SimpleAllocator::Deallocate

The last method is `Deallocate()`:

```cpp
void SimpleAllocator::Deallocate(void *ptr) noexcept {
  if (!ptr) { return; }

  auto *memory_block = MemoryBlock::FromUserMemory(ptr);
  if (memory_block->UserMemoryEnd() == current_) {
    current_ = reinterpret_cast<uint8_t *>(memory_block);
  }
}
```

In this implementation, the allocator is able to deallocate memory blocks that are only adjusted to the main buffer.

![MemoryBlock deallocation](/assets/img/posts/crafting-memory-allocators/5-deallocate.png)
_MemoryBlock deallocation_

But what should be done with others? For example:

```cpp
auto *p1 = allocator.Allocate(10);
auto *p2 = allocator.Allocate(12);
allocator.Deallocate(p1);
```

In the example, `p1` is not adjusted to the main buffer. So, what should we do with this and similar pieces of memory?

![MemoryBlock deallocation](/assets/img/posts/crafting-memory-allocators/6-deallocate.png)
_MemoryBlock deallocation_

Answering this question here, we will proceed to construct the entire allocator.
In a sense, an allocator deals with how to reuse the deallocated memory that cannot be returned to the main buffer.

## Slabs and Slots

There is a quite efficient and popular way to manage memory,
called [Slab allocation](https://en.wikipedia.org/wiki/Slab_allocation).
I will not use it as is, but I will use the idea.

1. Prepare a table which keeps the deallocated memory slots by their sizes.
2. The memory slots represent a forward linked list.
3. In `Deallocate()`, we put the deallocating `MemoryBlock` into the corresponding slot.
4. In `Allocate()`, we check if there is an available `MemoryBlock` in the corresponding slot and return it.

### MemorySlot

We need a class that represents memory slot list. Here it is:

```cpp
class MemorySlot {
public:
  constexpr MemorySlot() = default;

  void AddNext(MemoryBlock *memory_block) noexcept {
    next_ = new (memory_block->UserMemoryBegin()) MemorySlot{next_};
  }

  MemoryBlock *GetNext() noexcept {
    if (next_) {
      auto *next = next_;
      next_ = next->next_;
      return MemoryBlock::FromUserMemory(next);
    }
    return nullptr;
  }

private:
  constexpr explicit MemorySlot(MemorySlot *next) noexcept: next_{next} {}

  MemorySlot *next_{nullptr};
};
```

A few points about the class implementation:

1. The `MemorySlot` object is a list node itself.
2. `MemorySlot::AddNext()` constructs the next node using deallocated user memory. Since every `MemoryBlock`
has at least `SimpleAllocatorTraits::ALIGNMENT` available bytes (after size alignment) of user memory,
we can use them to place the `MemorySlot` object there.
3. `MemorySlot::GetNext()` retrieves the next available slot and casts it back to a `MemoryBlock`.
4. `MemoryBlock` is not destroyed after being added, and it keeps valid metadata.

![MemorySlot list](/assets/img/posts/crafting-memory-allocators/7-slots.png)
_MemorySlot list_

### SimpleAllocator with MemorySlot

Now, let's add a `MemorySlot` table to the `SimpleAllocator` class:

```cpp
class SimpleAllocatorBase {
private:
  static constexpr size_t ConstExprLog2(size_t x) {
    return x == 1 ? 0 : 1 + ConstExprLog2(x >> 1);
  }

protected:
  static constexpr size_t GetSlotIndex(size_t size) noexcept {
    constexpr size_t alignment_shift = ConstExprLog2(SimpleAllocatorTraits::ALIGNMENT);
    return (size >> alignment_shift) - 1;
  }
};

class SimpleAllocator : SimpleAllocatorBase {
public:
  // ...
private:
  // ...
  constexpr static size_t MAX_SLOT_SIZE_{16 * 1024};
  std::array<MemorySlot, GetSlotIndex(MAX_SLOT_SIZE_)> slots_{};
};
```

Why do we need `SimpleAllocatorBase`? Because in C++, it is not permitted to call static constexpr methods for
declarations within the same class.
As you may have noticed, `GetSlotIndex()` is called to calculate the size for the slot table array. Therefore, defining
the method in a base class serves as a good workaround for calling a constexpr method.

Let's take a closer look at the method `GetSlotIndex()`. This method returns the index in the slot table where the
memory slot with the specific size could be retrieved.
It is assumed that the size passed into the method is aligned, meaning it is not less
than `SimpleAllocatorTraits::ALIGNMENT` (16 in our case).
It is much easier to understand if we take a look at some examples:

| call                      | returns |
|---------------------------|---------|
| `GetSlotIndex(16)`        | `0`     |
| `GetSlotIndex(32)`        | `1`     |
| `GetSlotIndex(48)`        | `2`     |
| `GetSlotIndex(1024)`      | `63`    |
| `GetSlotIndex(16 * 1024)` | `1023`  |

So, `SimpleAllocator::slots_` represents the slot table with available `MemoryBlock` objects.

![The slot table](/assets/img/posts/crafting-memory-allocators/8-slot-table.png)
_The slot table_

There is an important aspect to note: the table keeps slots with a size less than `SimpleAllocator::MAX_SLOT_SIZE_`
bytes. We will discuss later what this means for the allocator.

### SimpleAllocator::Deallocate with MemorySlot

Let's use the memory slot table in the `Deallocate()` method.

```cpp
void SimpleAllocator::Deallocate(void *ptr) noexcept {
  // ...
  auto *memory_block = MemoryBlock::FromUserMemory(ptr);
  // ...
  // If possible, extend the main buffer by the memory block and return
  // ...
  const size_t slot_index = GetSlotIndex(memory_block->GetBlockSize());
  if (slot_index < slots_.size()) {
    slots_[slot_index].AddNext(memory_block);
  }
}
```

So now if the memory block is not adjusted to the main buffer, we put it into the slot table.

### SimpleAllocator::Allocate with MemorySlot

The same idea applies to the `Allocate()` method: before checking the available size in the main buffer,
attempt to obtain a memory block from the slot table.

```cpp
void *SimpleAllocator::Allocate(size_t size) noexcept {
  // ...
  const size_t slot_index = GetSlotIndex(size);
  if (slot_index < slots_.size()) {
    if (MemoryBlock *memory_block = slots_[slot_index].GetNext()) {
      return memory_block->UserMemoryBegin();
    }
  }
  // ...
  // Allocate from the main buffer
  // ...
}
```

One question remains: the slot table has a fixed size, and we can only use it for storing memory blocks with sizes less
than `SimpleAllocator::MAX_SLOT_SIZE_` bytes. However, what should be done if the memory block size is greater than or
equal to `SimpleAllocator::MAX_SLOT_SIZE_` bytes?

## The memory tree

The slot table stores a list of memory blocks, all of which have the same specific size. Access to the list
is performed by an index, which can be easily calculated, enabling quick access to the required memory block.
Consequently, retrieving a memory block from the slot table costs almost nothing. However,
this approach has a disadvantage: the slot table has a fixed size, so we cannot keep all sizes in the table.
This means we need a data structure suited to the following requirements:

1. It can grow dynamically.
2. The nodes of the data structure can be constructed directly on a memory block (within freed user memory).
3. Fast searching for a memory block suitable for a specific size.

We may try to use a simple forward list; it perfectly fits the first two requirements.
However, the problem lies in searching for a suitable memory block in a forward list,
which is a linear operation with `O(N)` runtime, making it very slow.

Instead, we may consider using a binary tree with the memory block size as a node key.
This structure can be dynamically grown, and tree nodes can be constructed using deallocated memory. Most importantly,
we can perform lookups for the suitable memory block with a runtime of `O(log(N))`, which is efficient.

![The memory tree](/assets/img/posts/crafting-memory-allocators/9-tree.png)
_The memory tree_

Additionally, this tree could be optimized.
We could use a tree node as a head for a linked list that stores nodes with the same sizes.

![The memory tree combined with linked lists](/assets/img/posts/crafting-memory-allocators/10-tree-optimized.png)
_The memory tree combined with linked lists_

Using a binary tree requires attention to tree balancing, as lookup in an unbalanced tree is a linear operation.
There are several methods for balancing a tree, such as

- Red-Black trees,
- AVL trees,
- and others.

AVL trees and Red-Black trees are very similar. The main differences are that AVL trees have faster search speed
because they are more balanced where Red-Black trees have better insert and remove speed.
During allocation, we need to search for the node and remove it from the tree.
During deallocation, we need to search for the parent and insert the node into the tree.
Since insertion and removal are performed with every tree access, I decided to use a Red-Black tree.
However, I haven't conducted any benchmarks; therefore, feel free to try an AVL tree or any other alternatives.

Implementing a Red-Black tree or an AVL tree can be challenging.
However, you can easily find many ready-made implementations on the internet,
including on Wikipedia: [Red-black tree](https://en.wikipedia.org/wiki/Red%E2%80%93black_tree)
and [AVL tree](https://en.wikipedia.org/wiki/AVL_tree).

Here is the Red-Black tree implementation for **The Simple Allocator**:
[MemoryTree.h](https://github.com/AlexK0/simple-allocator/blob/main/src/simple-allocator/MemoryTree.h) and
[MemoryTree.cpp](https://github.com/AlexK0/simple-allocator/blob/main/src/simple-allocator/MemoryTree.cpp).
I do not post the full implementation here since it consists of hundreds of lines of non-trivial code,
which isn't directly related to the allocator.

### MemoryTree

Let's start with the `MemoryTree::TreeNode` class, which represents a Red-Black tree node.

```cpp
class MemoryTree::TreeNode {
public:
  TreeNode *left{nullptr};
  TreeNode *right{nullptr};
  TreeNode *parent{nullptr};
  enum { RED, BLACK } color{RED};
  size_t block_size{0};
  TreeNode *same_size_nodes{nullptr};

  // methods..
};
```

Skipping the fields related to the tree, we're interested in two other fields:

- `MemoryTree::TreeNode::block_size`: the size of the memory block associated with this node.
- `MemoryTree::TreeNode::same_size_nodes`: a linked list that stores nodes with the same sizes.

> We could technically remove `MemoryTree::TreeNode::block_size` and utilize the size from `MemoryBlock::metadata`,
> but this would overcomplicate the code without providing any benefits.
> Therefore, I decided to duplicate the size into the tree node.
{: .prompt-info }

Here are the public methods of the class `MemoryTree`:

```cpp
class MemoryTree {
public:
  void InsertBlock(MemoryBlock *memory_block) noexcept;
  MemoryBlock *RetrieveBlock(size_t size) noexcept;

private:
  // ...
};
```

### MemoryTree::InsertBlock

The `MemoryTree::InsertBlock` method inserts a deallocated memory block into the tree. Here is the implementation:

```cpp
void MemoryTree::InsertBlock(MemoryBlock *memory_block) noexcept {
  const size_t size = memory_block->GetBlockSize();
  assert(sizeof(TreeNode) <= size);
  TreeNode *new_node = new (memory_block->UserMemoryBegin()) TreeNode{size};
  if (!root_) {
    new_node->color = TreeNode::BLACK;
    root_ = new_node;
    return;
  }

  TreeNode *parent = LookupNode(size, false);
  if (parent->block_size == size) {
    new_node->same_size_nodes = parent->same_size_nodes;
    parent->same_size_nodes = new_node;
    return;
  }

  new_node->parent = parent;
  if (size < parent->block_size) {
    parent->left = new_node;
  } else {
    parent->right = new_node;
  }

  FixRedRed(new_node);
}
```

Let's highlight the most interesting lines.

- Create a `TreeNode` using deallocated user memory for insertion into the tree.

```cpp
TreeNode *new_node = new (memory_block->UserMemoryBegin()) TreeNode{size};
```

- If the memory block has the same size as the parent node, add the new node to the linked list.

```cpp
if (parent->block_size == size) {
  new_node->same_size_nodes = parent->same_size_nodes;
  parent->same_size_nodes = new_node;
  return;
}
```

### MemoryTree::RetrieveBlock

The `MemoryTree::RetrieveBlock` method searches for a tree node associated with a memory block _suitable_ for
the requested size. Note that _suitable_ means the block size should be greater than or equal to the requested size.
In other words, we perform a lower-bound search.
If the node is found, it is removed from the tree and returned as a memory block.
Here is the implementation:

```cpp
MemoryBlock *MemoryTree::RetrieveBlock(size_t size) noexcept {
  if (TreeNode *v = LookupNode(size, true)) {
    if (v->same_size_nodes) {
      TreeNode *same_size_node = v->same_size_nodes;
      v->same_size_nodes = same_size_node->same_size_nodes;
      return MemoryBlock::FromUserMemory(same_size_node);
    }
    DetachNode(v);
    return MemoryBlock::FromUserMemory(v);
  }
  return nullptr;
}
```

One small addition: before removing the found node, we should check the node linked list.
If it contains another node with the same size, we retrieve the node from the linked list rather than from the tree.

### SimpleAllocator with MemoryTree

Now it's time to use `MemoryTree` in `SimpleAllocator`. Just add a new `MemoryTree` field to the class.

```cpp
class SimpleAllocator : SimpleAllocatorBase {
public:
  // ...
private:
  // ...
  MemoryTree memory_tree_;
}
```

### SimpleAllocator::Deallocate with MemoryTree

Let's use the memory tree in the `Deallocate()` method.

```cpp
void SimpleAllocator::Deallocate(void *ptr) noexcept {
  // ...
  auto *memory_block = MemoryBlock::FromUserMemory(ptr);
  // ...
  // If possible, extend the main buffer by the memory block and return
  // If possible, add the memory block to the slot table and return
  // ...
  memory_tree_.InsertBlock(memory_block);
}
```

It is pretty straightforward - if we cannot extend the main buffer by the deallocating memory block,
or if the deallocating memory block is too big for the slot table, we add it to the memory tree.

### SimpleAllocator::Allocate with MemoryTree

The `Allocate()` method becomes more complicated.

```cpp
void *SimpleAllocator::Allocate(size_t size) noexcept {
  // ...
  if (slot_index < slots_.size()) {
    // ...
    // Try to get the memory block from the slot table
    // ...
  } else if (MemoryBlock *memory_block = memory_tree_.RetrieveBlock(size)) {
    const size_t total_left_size = memory_block->GetBlockSize() - size;
    if (total_left_size > sizeof(MemoryBlock)) {
      const size_t user_left_size = total_left_size - sizeof(MemoryBlock);
      if (GetSlotIndex(user_left_size) >= slots_.size()) {
        memory_block->SetBlockSize(size);
        auto left_memory_block = new (memory_block->UserMemoryEnd()) MemoryBlock{user_left_size};
        memory_tree_.InsertBlock(left_memory_block);
      }
    }
    return memory_block->UserMemoryBegin();
  }

  // ...
  // Allocate from the main buffer
  // ...
}
```

Let's try to understand what's going on here. As mentioned earlier, `MemoryTree::RetrieveBlock` returns the
memory block using a lower bound lookup. This means that the found memory block has the requested size or is greater.
For example, if the memory tree contains only two memory blocks with sizes of `1MB` and `10MB`,
and we need to allocate `2MB`, `MemoryTree::RetrieveBlock` will return the `10MB` memory block.
We could return the `10MB` memory block to the user, but it would be excessive since the user only needs `2MB`.
Instead, we update the memory block to the requested size, trim the excess, and insert the trimmed portion
back into the memory tree.

![Split the memory block](/assets/img/posts/crafting-memory-allocators/11-split-memory-block.png)
_Split the memory block_

Another consideration is that we do not trim the memory block if the trimmed portion is not intended to be stored in
the memory tree. This is done in order to avoid unnecessary memory fragmentation.

For example, if the user requests `16400` bytes, and we retrieve a memory block for `16464` bytes,
after trimming the excess (taking into account the metadata size), we will have a memory block for
`48` bytes (`16464` - `16400` - `sizeof(MemoryBlock::metadata)`).
This block is meant to be stored in the slot table. Instead of inserting it into the slot table,
we leave the memory block for `16464` bytes as is and return it to the user.

That's all. The allocator is now complete. Now it's benchmarking time!

## Benchmarks

I don't want to compare bare `std::malloc`, `std::realloc`, and `std::free` with `SimpleAllocator` because
`SimpleAllocator` will be much faster in such benchmarks.
Instead, I want to compare how the allocator affects STL containers.

For benchmarking, I completely replace the system malloc family.
Since I have a macOS, for replacement, I use [dyld interposing](https://opensource.apple.com/source/dyld/dyld-210.2.3/include/mach-o/dyld-interposing.h).
I do not post the code here because it is not directly related to the topic, and this trick is specific to macOS.
However, if you're interested in how it is done, you can find the details in
[Malloc.cpp](https://github.com/AlexK0/simple-allocator/blob/main/src/malloc-replacement/Malloc.cpp#L132).

> The benchmarks were conducted on a MacBook Pro 2019 with a 2.3 GHz 8-Core Intel Core i9 processor.
{: .prompt-info }

> Keep in mind that the benchmark plots use a logarithmic scale for both the time and element axes.
{: .prompt-info }

> Benchmark plots are interactive; you may hide any scenario if you want.
{: .prompt-info }

### std::vector\<int64_t>

Let's compare [`push_back()`](https://github.com/AlexK0/simple-allocator/blob/main/src/benchmarks/Vector.cpp).

<html lang="en">
<head>
    <title>std::vector</title>
    <script src="/assets/js/chart.js"></script>
</head>
<body>
    <canvas id="vectorChart" width="400" height="300"></canvas>
    <script>
        var ctx = document.getElementById('vectorChart').getContext('2d');
        var myChart = new Chart(ctx, {
            type: 'line',
            data: {
                labels: ["1024", "2048", "4096", "8192", "16384", "32768"],
                datasets: [
                    {
                        label: 'SimpleAllocator: push_back()',
                        data: [1.88451, 2.3759, 3.27618, 5.21985, 9.41219, 18.6827],
                        borderColor: 'rgb(110,255,100)',
                        borderWidth: 3,
                        tension: 0.2
                    },
                    {
                        label: 'System malloc: push_back()',
                        data: [2.64391, 3.22137, 4.24968, 6.31922, 10.7325, 19.6813],
                        borderColor: 'rgb(255,155,25)',
                        borderWidth: 3,
                        tension: 0.2
                    },
                ]
            },
            options: {
                plugins: { title: { display: true, text: 'std::vector<int64_t>' } },
                scales: {
                    myScale: {
                        type: 'logarithmic',
                        position: 'left',
                        title: { display: true, text: 'Time in us (logarithmic)' }
                    },
                    x: { title: { display: true, text: 'Elements (logarithmic)' } }
                }
            }
        });
    </script>
</body>
</html>

| Elements | push_back()        |
|----------|--------------------|
| 1024     | 1.8 us  \| 2.6 us  |
| 2048     | 2.3 us  \| 3.2 us  |
| 4096     | 3.2 us  \| 4.2 us  |
| 8192     | 5.2 us  \| 6.3 us  |
| 16384    | 9.4 us  \| 10.7 us |
| 32768    | 18.6 us \| 19.6 us |

`std::vector::push_back()` with `SimpleAllocator` works a bit faster.
This difference is not too big because `std::vector::push_back()` has an amortized constant runtime,
which means it doesn't reallocate the internal array on each `push_back()`.
That's why even such a small performance boost looks good.

### std::deque\<int64_t>

Let's take a look at `std::deque<int64_t>`. I made [several benchmark scenarios](https://github.com/AlexK0/simple-allocator/blob/main/src/benchmarks/Deque.cpp).

<html lang="en">
<head>
    <title>std::deque</title>
    <script src="/assets/js/chart.js"></script>
</head>
<body>
    <canvas id="dequeChart" width="400" height="300"></canvas>
    <script>
        var ctx = document.getElementById('dequeChart').getContext('2d');
        var myChart = new Chart(ctx, {
            type: 'line',
            data: {
                labels: ["1024", "2048", "4096", "8192", "16384", "32768"],
                datasets: [
                    {
                        label: 'SimpleAllocator: push_back()',
                        data: [2.49756, 3.60143, 5.88526, 10.3772, 19.2391, 36.662],
                        borderColor: 'rgb(110,255,100)',
                        borderWidth: 3,
                        tension: 0.2
                    },
                    {
                        label: 'SimpleAllocator: pop_front()',
                        data: [1.65319, 2.01766, 2.74953, 4.04596, 6.87529, 12.582],
                        borderColor: 'rgb(100,208,255)',
                        borderWidth: 3,
                        tension: 0.2
                    },
                    {
                        label: 'SimpleAllocator: push_back() + pop_front()',
                        data: [3.12305, 4.46908, 7.5713, 13.9564, 25.7173, 50.5537],
                        borderColor: 'rgb(193,100,255)',
                        borderWidth: 3,
                        tension: 0.2
                    },
                    {
                        label: ' System malloc: push_back()',
                        data: [3.06354, 4.6784, 7.96355, 14.2529, 27.1116, 53.445],
                        borderColor: 'rgb(255,155,25)',
                        borderWidth: 3,
                        tension: 0.2
                    },
                    {
                        label: ' System malloc: pop_front()',
                        data: [1.86359, 2.51932, 3.8321, 6.29252, 11.2734, 21.0098],
                        borderColor: 'rgb(255,25,25)',
                        borderWidth: 3,
                        tension: 0.2
                    },
                    {
                        label: ' System malloc: push_back() + pop_front()',
                        data: [3.43679, 5.50381, 9.69749, 18.267, 35.2927, 68.8344],
                        borderColor: 'rgb(255,251,25)',
                        borderWidth: 3,
                        tension: 0.2
                    },
                ]
            },
            options: {
                plugins: { title: { display: true, text: 'std::deque<int64_t>' } },
                scales: {
                    myScale: {
                        type: 'logarithmic',
                        position: 'left',
                        title: { display: true, text: 'Time in us (logarithmic)' }
                    },
                    x: { title: { display: true, text: 'Elements (logarithmic)' } }
                }
            }
        });
    </script>
</body>
</html>

| Elements | push_back()          | pop_front()        | push_back() + pop_front() |
|----------|----------------------|--------------------|---------------------------|
| 1024     | 2.4 us  \| 3.0 us    | 1.6 us  \| 1.8 us  | 3.1 us  \| 3.4 us         |
| 2048     | 3.6 us  \| 4.6 us    | 2.0 us  \| 2.5 us  | 4.4 us  \| 5.5 us         |
| 4096     | 5.8 us  \| 7.9 us    | 2.7 us  \| 3.8 us  | 7.5 us  \| 9.6 us         |
| 8192     | 10.3 us \| 14.2 us   | 4.0 us  \| 6.2 us  | 13.9 us \| 18.2 us        |
| 16384    | 19.2 us \| 27.1 us   | 6.8 us  \| 11.2 us | 25.7 us \| 35.2 us        |
| 32768    | 36.6 us \| 53.4 us   | 12.5 us \| 21.0 us | 50.5 us \| 68.8 us        |

The same story applies as with `std::vector<int64_t>` — there is a small performance boost because
`std::deque<int64_t>` doesn't reallocate the internal storage on every `push_back()` and
doesn't deallocate the internal storage on every `pop_front()`.

### std::list\<int64_t>

The next benchmark is for `std::list<int64_t>`. It involves [the same scenarios](https://github.com/AlexK0/simple-allocator/blob/main/src/benchmarks/List.cpp) as for `std::deque<int64_t>`.

<html lang="en">
<head>
    <title>std::list</title>
    <script src="/assets/js/chart.js"></script>
</head>
<body>
    <canvas id="listChart" width="400" height="300"></canvas>
    <script>
        var ctx = document.getElementById('listChart').getContext('2d');
        var myChart = new Chart(ctx, {
            type: 'line',
            data: {
                labels: ["1024", "2048", "4096", "8192", "16384", "32768"],
                datasets: [
                    {
                        label: 'SimpleAllocator: push_back()',
                        data: [6.99511, 12.5524, 24.1294, 51.109, 93.0427, 186.925],
                        borderColor: 'rgb(110,255,100)',
                        borderWidth: 3,
                        tension: 0.2
                    },
                    {
                        label: 'SimpleAllocator: pop_front()',
                        data: [5.93903, 10.8639, 21.3376, 41.049, 85.472, 148.579],
                        borderColor: 'rgb(100,208,255)',
                        borderWidth: 3,
                        tension: 0.2
                    },
                    {
                        label: 'SimpleAllocator: push_back() + pop_front()',
                        data: [8.68775, 16.3359, 33.5982, 68.9303, 152.433, 348.106],
                        borderColor: 'rgb(193,100,255)',
                        borderWidth: 3,
                        tension: 0.2
                    },
                    {
                        label: ' System malloc: push_back()',
                        data: [26.3801, 51.8859, 103.481, 203.739, 408.913, 812.914],
                        borderColor: 'rgb(255,155,25)',
                        borderWidth: 3,
                        tension: 0.2
                    },
                    {
                        label: ' System malloc: pop_front()',
                        data: [34.0431, 67.9025, 134.968, 274.043, 547.713, 1084.27],
                        borderColor: 'rgb(255,25,25)',
                        borderWidth: 3,
                        tension: 0.2
                    },
                    {
                        label: ' System malloc: push_back() + pop_front()',
                        data: [39.9434, 78.4393, 156.884, 316.428, 638.825, 1276.49],
                        borderColor: 'rgb(255,251,25)',
                        borderWidth: 3,
                        tension: 0.2
                    },
                ]
            },
            options: {
                plugins: { title: { display: true, text: 'std::list<int64_t>' } },
                scales: {
                    myScale: {
                        type: 'logarithmic',
                        position: 'left',
                        title: { display: true, text: 'Time in us (logarithmic)' }
                    },
                    x: { title: { display: true, text: 'Elements (logarithmic)' } }
                }
            }
        });
    </script>
</body>
</html>

| Elements | push_back()          | pop_front()           | push_back() + pop_front() |
|----------|----------------------|-----------------------|---------------------------|
| 1024     | 6.9 us   \| 26.3 us  | 5.9 us   \| 34.0 us   | 8.6 us   \| 39.9 us       |
| 2048     | 12.5 us  \| 51.8 us  | 10.8 us  \| 67.9 us   | 16.3 us  \| 78.4 us       |
| 4096     | 24.1 us  \| 103.4 us | 21.3 us  \| 134.9 us  | 33.5 us  \| 156.8 us      |
| 8192     | 51.1 us  \| 203.7 us | 41.0 us  \| 274.0 us  | 68.9 us  \| 316.4 us      |
| 16384    | 93.0 us  \| 408.9 us | 85.4 us  \| 547.7 us  | 152.4 us \| 638.8 us      |
| 32768    | 186.9 us \| 812.9 us | 148.5 us \| 1084.2 us | 348.1 us \| 1276.4 us     |

Well, now the difference is much more visible. The benchmarks reveal that methods of `std::list<int64_t>` with
`SimpleAllocator` are almost 5 times faster compared to using the system malloc. This occurs because
`std::list<int64_t>`, unlike `std::vector<int64>` and `std::deque<int64>`, performs an allocation on each `push_back()`
call and a deallocation on each `pop_front()` call.

### std::list\<std::string>

Let's conduct
[another benchmark](https://github.com/AlexK0/simple-allocator/blob/main/src/benchmarks/ListHugeElement.cpp),
but this time, instead of simple `int64_t`, we'll use large `std::string` objects with sizes greater than `16KB`.
This will prompt `SimpleAllocator` to utilize the memory tree.

<html lang="en">
<head>
    <title>std::list std::string</title>
    <script src="/assets/js/chart.js"></script>
</head>
<body>
    <canvas id="listStringChart" width="400" height="300"></canvas>
    <script>
        var ctx = document.getElementById('listStringChart').getContext('2d');
        var myChart = new Chart(ctx, {
            type: 'line',
            data: {
                labels: ["1024", "2048", "4096", "8192", "16384", "32768"],
                datasets: [
                    {
                        label: 'SimpleAllocator: push_back()',
                        data: [543.274, 1727.3, 3770.59, 8599.22, 21685.6, 70585.9],
                        borderColor: 'rgb(110,255,100)',
                        borderWidth: 3,
                        tension: 0.2
                    },
                    {
                        label: 'SimpleAllocator: pop_front()',
                        data: [30.9069, 135.528, 328.478, 696.627, 1455.65, 2985.31],
                        borderColor: 'rgb(100,208,255)',
                        borderWidth: 3,
                        tension: 0.2
                    },
                    {
                        label: 'SimpleAllocator: push_back() + pop_front()',
                        data: [548.161, 1885.83, 4209.94, 10105.8, 24767, 84584.5],
                        borderColor: 'rgb(193,100,255)',
                        borderWidth: 3,
                        tension: 0.2
                    },
                    {
                        label: ' System malloc: push_back()',
                        data: [3569.24, 9743.81, 23369.3, 58418.5, 120360, 268069],
                        borderColor: 'rgb(255,155,25)',
                        borderWidth: 3,
                        tension: 0.2
                    },
                    {
                        label: ' System malloc: pop_front()',
                        data: [1504.13, 3196.79, 5267.29, 12642.3, 28169.7, 70063.7],
                        borderColor: 'rgb(255,25,25)',
                        borderWidth: 3,
                        tension: 0.2
                    },
                    {
                        label: ' System malloc: push_back() + pop_front()',
                        data: [1031.81, 4147.53, 13091, 32339.7, 74833.7, 168244],
                        borderColor: 'rgb(255,251,25)',
                        borderWidth: 3,
                        tension: 0.2
                    },
                ]
            },
            options: {
                plugins: { title: { display: true, text: 'std::list<std::string>' } },
                scales: {
                    myScale: {
                        type: 'logarithmic',
                        position: 'left',
                        title: { display: true, text: 'Time in us (logarithmic)' }
                    },
                    x: { title: { display: true, text: 'Elements (logarithmic)' } }
                }
            }
        });
    </script>
</body>
</html>

| Elements | push_back()               | pop_front()             | push_back() + pop_front() |
|----------|---------------------------|-------------------------|---------------------------|
| 1024     | 543.2 us   \| 3569.2 us   | 30.9 us   \| 1504.1 us  | 548.1 us   \| 1031.8 us   |
| 2048     | 1727.3 us  \| 9743.8 us   | 135.5 us  \| 3196.7 us  | 1885.8 us  \| 4147.5 us   |
| 4096     | 3770.5 us  \| 23369.3 us  | 328.4 us  \| 5267.2 us  | 4209.9 us  \| 13091.0 us  |
| 8192     | 8599.2 us  \| 58418.5 us  | 696.6 us  \| 12642.3 us | 10105.8 us \| 32339.7 us  |
| 16384    | 21685.6 us \| 120360.0 us | 1455.6 us \| 28169.7 us | 24767.0 us \| 74833.7 us  |
| 32768    | 70585.9 us \| 268069.0 us | 2985.3 us \| 70063.7 us | 84584.5 us \| 168244.0 us |

Now the difference is much more visible due to an additional allocation for copying a string and an extra deallocation
for destroying a string. It seems that the memory tree works pretty fast.

### std::map\<int64_t, int64_t>

Another interesting case involves associative containers. I made
[different scenarios for them](https://github.com/AlexK0/simple-allocator/blob/main/src/benchmarks/Map.cpp),
so let's take a look.

<html lang="en">
<head>
    <title>std::map</title>
    <script src="/assets/js/chart.js"></script>
</head>
<body>
    <canvas id="mapChart" width="400" height="300"></canvas>
    <script>
        var ctx = document.getElementById('mapChart').getContext('2d');
        var myChart = new Chart(ctx, {
            type: 'line',
            data: {
                labels: ["1024", "2048", "4096", "8192", "16384", "32768"],
                datasets: [
                    {
                        label: ' SimpleAllocator: insert()',
                        data: [28.8932, 60.375, 127.876, 268.108, 568.808, 1222.85],
                        borderColor: 'rgb(110,255,100)',
                        borderWidth: 3,
                        tension: 0.2
                    },
                    {
                        label: ' SimpleAllocator: erase()',
                        data: [16.9345, 31.954, 64.0645, 122.485, 245.756, 522.787],
                        borderColor: 'rgb(100,208,255)',
                        borderWidth: 3,
                        tension: 0.2
                    },
                    {
                        label: ' SimpleAllocator: insert() + erase()',
                        data: [44.5758, 99.7168, 216.321, 473.799, 1020.55, 2218.18],
                        borderColor: 'rgb(193,100,255)',
                        borderWidth: 3,
                        tension: 0.2
                    },
                    {
                        label: '  System malloc: insert()',
                        data: [49.2182, 102.053, 221.353, 490.636, 1002.47, 2354.82],
                        borderColor: 'rgb(255,155,25)',
                        borderWidth: 3,
                        tension: 0.2
                    },
                    {
                        label: '  System malloc: erase()',
                        data: [57.1825, 111.106, 221.526, 439.599, 801.954, 1564.62],
                        borderColor: 'rgb(255,25,25)',
                        borderWidth: 3,
                        tension: 0.2
                    },
                    {
                        label: '  System malloc: insert() + erase()',
                        data: [79.2123, 166.39, 360.172, 763.323, 1573.75, 3261.7],
                        borderColor: 'rgb(255,251,25)',
                        borderWidth: 3,
                        tension: 0.2
                    },
                ]
            },
            options: {
                plugins: { title: { display: true, text: 'std::map<int64_t, int64_t>' } },
                scales: {
                    myScale: {
                        type: 'logarithmic',
                        position: 'left',
                        title: { display: true, text: 'Time in us (logarithmic)' }
                    },
                    x: { title: { display: true, text: 'Elements (logarithmic)' } }
                }
            }
        });
    </script>
</body>
</html>

| Elements | insert()               | erase()               | insert() + erase()     |
|----------|------------------------|-----------------------|------------------------|
| 1024     | 28.8 us   \| 49.2 us   | 16.9 us  \| 57.1 us   | 44.5 us   \| 79.2 us   |
| 2048     | 60.3 us   \| 102.0 us  | 31.9 us  \| 111.1 us  | 99.7 us   \| 166.3 us  |
| 4096     | 127.8 us  \| 221.3 us  | 64.0 us  \| 221.5 us  | 216.3 us  \| 360.1 us  |
| 8192     | 268.1 us  \| 490.6 us  | 122.4 us \| 439.5 us  | 473.7 us  \| 763.3 us  |
| 16384    | 568.8 us  \| 1002.4 us | 245.7 us \| 801.9 us  | 1020.5 us \| 1573.7 us |
| 32768    | 1222.8 us \| 2354.8 us | 522.7 us \| 1564.6 us | 2218.1 us \| 3261.7 us |

`std::map<int64_t, int64_t>` behaves similarly to `std::list<int64_t>`: with every `insert()`, it performs
an allocation, and with every `erase()`, it performs a deallocation. The difference lies in the additional tree
operations of `std::map<int64_t, int64_t>`, which have a runtime of `O(log(N))`. Consequently, the performance
improvement is less significant compared to `std::list<int64_t>`. However, `std::map<int64_t, int64_t>` with
`SimpleAllocator` still exhibits faster performance, which is favorable.

### std::unordered_map\<int64_t, int64_t>

This is the final benchmark for `SimpleAllocator`, where I use `std::unordered_map<int64_t, int64_t>`,
[with scenarios similar](https://github.com/AlexK0/simple-allocator/blob/main/src/benchmarks/UnorderedMap.cpp)
to those of `std::map<int64_t, int64_t>`.

<html lang="en">
<head>
    <title>std::unordered_map</title>
    <script src="/assets/js/chart.js"></script>
</head>
<body>
    <canvas id="unorderedMapChart" width="400" height="300"></canvas>
    <script>
        var ctx = document.getElementById('unorderedMapChart').getContext('2d');
        var myChart = new Chart(ctx, {
            type: 'line',
            data: {
                labels: ["1024", "2048", "4096", "8192", "16384", "32768"],
                datasets: [
                    {
                        label: ' SimpleAllocator: insert()',
                        data: [17.5577, 35.9626, 69.0099, 147.154, 300.357, 569.616],
                        borderColor: 'rgb(110,255,100)',
                        borderWidth: 3,
                        tension: 0.2
                    },
                    {
                        label: ' SimpleAllocator: erase()',
                        data: [10.7147, 19.8562, 37.4061, 68.6887, 143.583, 265.482],
                        borderColor: 'rgb(100,208,255)',
                        borderWidth: 3,
                        tension: 0.2
                    },
                    {
                        label: ' SimpleAllocator: insert() + erase()',
                        data: [28.2267, 55.0477, 114.764, 219.751, 442.763, 908.823],
                        borderColor: 'rgb(193,100,255)',
                        borderWidth: 3,
                        tension: 0.2
                    },
                    {
                        label: '  System malloc: insert()',
                        data: [42.5049, 80.8013, 153.878, 331.393, 606.063, 1282.32],
                        borderColor: 'rgb(255,155,25)',
                        borderWidth: 3,
                        tension: 0.2
                    },
                    {
                        label: '  System malloc: erase()',
                        data: [38.903, 78.6209, 162.551, 357.892, 674.399, 1327.12],
                        borderColor: 'rgb(255,25,25)',
                        borderWidth: 3,
                        tension: 0.2
                    },
                    {
                        label: '  System malloc: insert() + erase()',
                        data: [61.6733, 127.195, 233.997, 492.677, 975.703, 1928.04],
                        borderColor: 'rgb(255,251,25)',
                        borderWidth: 3,
                        tension: 0.2
                    },
                ]
            },
            options: {
                plugins: { title: { display: true, text: 'std::unordered_map<int64_t, int64_t>' } },
                scales: {
                    myScale: {
                        type: 'logarithmic',
                        position: 'left',
                        title: { display: true, text: 'Time in us (logarithmic)' }
                    },
                    x: { title: { display: true, text: 'Elements (logarithmic)' } }
                }
            }
        });
    </script>
</body>
</html>

| Elements | insert()              | erase()               | insert() + erase()    |
|----------|-----------------------|-----------------------|-----------------------|
| 1024     | 17.5 us  \| 42.5 us   | 10.7 us \| 38.9 us    | 28.2 us  \| 61.6 us   |
| 2048     | 35.9 us  \| 80.8 us   | 19.8 us \| 78.6 us    | 55.0 us  \| 127.1 us  |
| 4096     | 69.0 us  \| 153.8 us  | 37.4 us \| 162.5 us   | 114.7 us \| 233.9 us  |
| 8192     | 147.1 us \| 331.3 us  | 68.6 us \| 357.8 us   | 219.7 us \| 492.6 us  |
| 16384    | 300.3 us \| 606.0 us  | 143.5 us \| 674.3 us  | 442.7 us \| 975.7 us  |
| 32768    | 569.6 us \| 1282.3 us | 265.4 us \| 1327.1 us | 908.8 us \| 1928.0 us |

The performance boost is more significant than in the `std::map<int64_t, int64_t>` benchmark because the hash table
operations have better runtimes, and they affect the `insert()` and `erase()` methods less.

## What's next?

Well, the benchmarks show that using `SimpleAllocator` gives a performance boost, and this is great.
However, there are several features that I didn't implement to avoid overcomplexity.
Nevertheless, it is worth considering adding these features to `SimpleAllocator`.

### Metrics

I didn't add any metrics to `SimpleAllocator`, but it would be very handy to know
- the main buffer size,
- how much memory is allocated,
- how much memory is available,
- how much memory is wasted due to metadata and alignment,
- etc.

### Address sanitizer support

Using `SimpleAllocator` reduces the effectiveness of the
[Address Sanitizer](https://github.com/google/sanitizers/wiki/AddressSanitizer) because the allocator
operates with a preallocated buffer, making it impossible to perform certain Address Sanitizer checks,
such as heap buffer overflow, memory usage after free, double free, etc.

To mitigate these issues, we could use
[Address Sanitizer manual poisoning](https://github.com/google/sanitizers/wiki/AddressSanitizerManualPoisoning).
This feature allows making memory regions inaccessible. We could poison:
- memory around the main buffer,
- memory around a `MemoryBlock` object,
- `MemoryBlock::metadata` before returning the memory pointer to a user,
- unused user memory in a `MemoryBlock` object after storing it into the slot table or memory tree,
- etc.

### Memory defragmentation

Prolonged usage of `SimpleAllocator` may lead to fragmentation of the main buffer. This can result in the inability to
allocate a large contiguous memory block, even if the allocator has enough memory available, due to the memory being
split by the slot table and memory tree nodes. The simplest solution to this issue is to perform eventual
defragmentation. The algorithm is straightforward:

- collect all `MemoryBlock` objects from the slot table and memory tree,
- sort them by their addresses,
- and merge adjacent memory blocks.

Additionally, if we want to reuse the defragmented memory block as the main buffer, similar to the source of memory,
some logic around the main buffer should be updated. For example, we could introduce pointers to the active buffer and
use them instead of `current_`, `buffer_begin_`, and `buffer_end_`.

While this approach may not be optimal, it is functional and can extend the usage time of the allocator before it
starts returning `nullptr` due to out of memory conditions. Feel free to explore alternative solutions.

### Multithreading

Well, it is clear that `SimpleAllocator` cannot be used from different threads simultaneously. Of course, we could use
`std::mutex` and lock every access to `SimpleAllocator`, but this would make the allocator extremely slow.
Can it be better? Probably. The main buffer pointers can be replaced with atomics. The slot table lists can be replaced
with lock-free forward lists. However, I'm not sure what to do with the memory tree. You may find some implementations
of concurrent binary search trees on the internet, but I'm not sure if they can be adapted for `SimpleAllocator`.
Nevertheless, this is a very interesting challenge to solve. Feel free to try different approaches, but don't forget
about benchmarks; lock-free data structures come with their own costs.

Generally speaking, that's why the system malloc is less performant than `SimpleAllocator`, because it concerns itself
with memory fragmentation and works perfectly from different threads. `SimpleAllocator`, on the other hand, ignores
that and gains a performance benefit.

## References

- [The Simple Allocator implementation on github](https://github.com/AlexK0/simple-allocator).
  - [`SimpleAllocator` class](https://github.com/AlexK0/simple-allocator/tree/main/src/simple-allocator).
  - [System malloc replacer for macOS](https://github.com/AlexK0/simple-allocator/tree/main/src/malloc-replacement).
  - [Benchmarks](https://github.com/AlexK0/simple-allocator/tree/main/src/benchmarks).
  - [Tests](https://github.com/AlexK0/simple-allocator/tree/main/src/tests).
- [The Slab allocation article on Wikipedia](https://en.wikipedia.org/wiki/Slab_allocation).
- [The Red-black tree article on Wikipedia](https://en.wikipedia.org/wiki/Red%E2%80%93black_tree).
