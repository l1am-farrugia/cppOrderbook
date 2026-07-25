# Limit Orderbook Matching Engine (C++)

To understand how I built this matching engine, we first have to look at what an order book actually does at a conceptual level. At its core, an electronic order book is just a data structure that keeps track of buyers and sellers in a financial market. It has two main sides: 

- The Bids: A collection of buy orders, sorted from the highest price to the lowest price.

  (Buyers want to pay as little as possible, so the highest bid is the most competitive). 

- The Asks (or Offers): A collection of sell orders, sorted from the lowest price to the highest price. 

  (Sellers want to get paid as much as possible, so the lowest ask is the most competitive). 

The point where the highest bid and the lowest ask meet is the current market price. 

When a new order arrives, the engine has a simple choice. If the incoming order is a "Taker" order, meaning it crosses the market price to immediately match with a resting "Maker" order, the engine executes a trade. If it doesn't match, it becomes a Maker order itself, sitting in the book and waiting for someone else to fill it.

In my original implementation, the system was built using standard C++ containers. 

- `std::map<PriceTicks, PriceLevel>` was used to hold the bids and asks. 

- Each `PriceLevel` contained a `std::list<Order*>` to store individual orders at that specific price in a FIFO queue.

- A `std::unordered_map<OrderId, Locator>` acted as a master index so we could look up any order instantly by its ID when a user wanted to cancel it. 

When evaluated using the small benchmark script over 1,000,000 iterations, the original implementation achieved an execution time of **58 nanoseconds (ns)** on average per event. However, this metric represented an ideal scenario rather than a realistic workload. Because the script only maintained a small footprint of about a dozen active orders at any given moment, the standard library containers never grew to be very large. The entire execution footprint fits entirely inside the L1 cache, requiring minimal tree comparisons. To test on a much heavier script, I evaluated the engine against a dense market script `dense_script.txt` clustering 50,000 events tightly within a narrow 100-tick price window, and a highly sparse macro market script `deep_script.txt` scattering 20,000 active resting orders widely across a 200,000-tick spread. Under the dense workload, the original engine's average event time expanded to about **90ns**, generating **48 million L1 data cache misses**. Under the sparse macro workload, performance degraded to about **660ns** per event, thrashing the CPU with **4 Billion L1 cache misses**.

To understand why this architecture becomes slow at scale, we have to look at how the program interacts with the operating system and hardware layers, and what issues arise from those interactions which slow the program. The two main issues I found with the original implementation were: 

1) When the engine executes `new Order()`, the C++ runtime allocator searches its own user space free list to find a piece of memory on the heap that fits the object. If the allocator runs out of memory, it makes a system call to ask the OS for a new page, which can trigger a mode switch to kernel space. The OS ensures no other program can touch this memory by isolating it within your process's private virtual address space, after which the runtime constructs the object and passes it back to your code

   In normal applications this switching takes negligible time, but for the orderbook, in the critical loop where orders are being matched, making the CPU jump back and forth between our engine and the OS kernel is costly.

   Furthermore, `std::list` and `std::map` are node based containers. Every single time a new order enters the book or a price level is created, they technically call `new` under the hood to allocate a node. When an order is filled, they call `delete`. This constant memory churn creates massive, non-deterministic latency spikes.
  
2) The original orderbook also suffered from poor cache utilisation. System RAM has good capacity, but is physically located far away from the CPU core. Accessing it is (in this case) very slow, taking around 60-100ns on average. Alternatively, L1/L2 Caches are memory built directly onto the CPU chip itself. Accessing the L1 cache is very fast, taking less than 1ns (generally about 3-4 CPU clock cycles), while L2 is slightly slower at about 3-5ns or 10-20 clock cycles.
   The CPU reads data in cache lines, if the data the CPU needs next is already sitting in that cache line, we have a cache hit and the data is processed instantly. If the data is scattered somewhere else in RAM, it suffers a cache miss, and the entire CPU core completely stalls, sitting idle for about 100ns waiting for the data to arrive.

   Since our original design used `std::list` and `std::map`, where every node is allocated independently via `new`, the memory addresses end up completely scattered across the heap. Node A might be at address 0x1000, while Node B is at 0x9999. When the CPU tries to traverse the linked list to match orders, it reads Node A, follows the pointer to Node B, and instantly triggers a cache miss because Node B is nowhere near Node A in physical memory. Since the data has no predictable pattern, the CPU prefetcher becomes completely useless as it can’t easily predict what memory is needed next in order to load it ahead of time. `std::list` is thus terrible for our cache lines because it does exactly this.

   Using `std::list<Order*>`, created two layers of indirection. The standard library container allocates its own internal node structure under the hood. This node contains pointers to the next and previous list elements, along with a pointer to your actual `Order` struct.

   <img width="1335" height="476" alt="Image 1" src="https://github.com/user-attachments/assets/f52f8cc0-43c1-4509-8af0-02526e106aed" />

   To traverse this queue, the CPU has to read the list node, find the memory address of the order, jump to that address, and then jump back to the next list node. This constant ping-ponging across the system memory has a high chance of resulting in cache misses.

To fix this, I used an Intrusive Doubly-Linked List. Instead of letting an external container manage our layout, the `next` and `prev` pointers were embedded directly inside the `Order` struct itself:

```cpp
struct Order
{
    OrderId id;
    Side side;
    PriceTicks price_ticks;
    Qty qty;
    std::uint64_t seq;

    Order* next { nullptr };
    Order* prev { nullptr };
};
```

This change fundamentally reshapes how the data sits in memory. When the matching engine fetches an order, the pointers required to find the next order in the FIFO queue are loaded into the exact same cache line simultaneously.

<img width="1439" height="364" alt="Image 2" src="https://github.com/user-attachments/assets/27f46995-9faf-49d2-9399-4b0bccdb282b" />

This completely eliminated the mid-level container nodes. Now, each `PriceLevel` only needs to store a head and a tail pointer. Walking the queue becomes a straightforward matter of following the embedded pointers already sitting in the CPU's L1 cache.

While this cleaned up our memory layout, we were still fundamentally tied to the OS because `new` and `delete` were used to create and destroy the `Order` nodes. To remove the dynamic memory allocation completely, I built a custom memory pool. 

The core philosophy here is simple: 

*Allocate everything upfront and recycle everything aggressively*

Instead of asking the kernel for a small piece of memory every single time an order is submitted, the orderbook will pre-allocate a massive contiguous block of 4,096 `Order` structs at startup.

To manage this block without adding any extra memory overhead, I implemented a Free List. Because an idle order node isn't resting in the book, its embedded next pointer isn't being used, so we can repurpose that pointer to link all the unallocated slots together in a single chain.

<img width="1391" height="103" alt="Image 3" src="https://github.com/user-attachments/assets/389e8ddc-283d-47af-93da-4f0f2b6fda95" />

When a new limit order arrives at the matching engine, we call `allocate_order()`:

```cpp
Order* OrderBook::allocate_order()
{
    if (free_list_ != nullptr)
    {
        Order* o = free_list_;
        free_list_ = o->next;
        return o;
    }
}
```

This is an O(1) operation. The engine grabs the node right at the front of `free_list_`, shifts the pointer forward, and hands the memory back. This completely avoids any system calls and heap traversals.

When an order gets filled or cancelled, we pass it to `free_order()`:

```cpp
void OrderBook::free_order(Order* o)
{
    o->next = free_list_;
    free_list_ = o;
}
```

Instead of wiping the memory or releasing it back to the OS, the engine weaves the dead node right back into the front of the free list. The memory stays warm in the CPU cache, ready to be reused by the very next incoming event. 

By implementing the intrusive layout and the memory pool, the hot path is completely protected from the kernel, bringing our latency down to a stable baseline of **728ns**.

With the order nodes tightly packed via the memory pool and linked lists, the next step was to revise how the engine searched for price levels. 

Originally, I relied on `std::map<PriceTicks, PriceLevel>`. A C++ `std::map` is universally implemented as a Red-Black Tree. When the matching engine receives an incoming order, it has to look up the correct price level by starting at the root node of the tree and following left or right pointers down to the target leaf.

This is a massive performance bottleneck because of the pointer chasing. To find a price level, the CPU core has to read a node, extract a pointer address, jump to that new memory address, read the next node, and repeat. Even though the lookup complexity is O(log n), it forces the CPU to constantly guess where to look next, again disabling the prefetcher. 

In attempts to correct this, I initially stripped out the `std::map` completely and replaced it with `std::vector<LevelRecord>`. The vector stores all its elements in a single contiguous block of memory. To keep the price levels sorted (highest bids first, lowest asks first), I used `std::lower_bound` to run a binary search across the array whenever a new price level needed to be found. Because the vector is contiguous, the prefetcher can load adjacent elements into the L1 cache before the code even requests them, providing a cleaner memory access routine.

<img width="1405" height="176" alt="Image 4" src="https://github.com/user-attachments/assets/186d228c-6851-476f-ad78-76ccca423610" />

This however, barely improved execution time, from an average of **728ns** to **708ns**. While searching a vector is fast, inserting a new price level into the middle of the array forces the vector to move every subsequent element one slot down in memory to maintain order. This memory shuffle effectively cancelled out the benefits gained from avoiding the tree traversal.

I then implemented a Direct-Mapped Static Array to maintain the cache locality of a vector while avoiding binary searches and those memory shuffles. Instead of storing only active price levels, two arrays (one for bids and one for asks) are used and pre-sized to a fixed window of 1,000,000 price ticks.

```cpp
static constexpr std::int64_t MAX_TICKS = 1000000;
std::vector<PriceLevel> bids_(MAX_TICKS);
std::vector<PriceLevel> asks_(MAX_TICKS);
```

This fundamentally transformed how the engine resolves prices. The price tick value is the index of the array. If we send a buy order at a price tick of 420,069, the matching engine will calculate the memory offset instantly: `bids_[420069]`

Lookup and insertion are O(1). Removing the memory shuffles, pointer chasing, and sorting routines. The memory for every possible price level is allocated exactly once at system startup.

*“Of course, maintaining massive static arrays introduces a new engineering hurdle. If an incoming buy order needs to match against the best available ask price, how do we know where the top of the book is without looping through all one million slots?”*

Initially, I added logic to search the array sequentially whenever the best price level became empty, and introduced global tracking variables `best_bid_`, `best_ask_` alongside active order counters `active_bid_count_`, `active_ask_count_`

```cpp
if (active_ask_count_ == 0)
{
    best_ask_ = MAX_TICKS;
}
else if (level.empty() && loc.price_ticks == best_ask_)
{
    while (best_ask_ < MAX_TICKS && asks_[best_ask_].empty())
    {
        best_ask_++;
    }
}
```

By tracking live order quantities globally, I aimed to instantly short circuit empty price slots, allowing the pointer to step past blank ranges. 

While this successfully handled basic book clearings, it introduced something I called **“The Empty Book Scan of Doom and Despair”**. When exposed to a highly sparse macro market layout, active price clusters are separated by giant bulks of empty ticks. If a large aggressive order swept the top of the book, the engine was forced to step sequentially through thousands of inactive array indices using a linear pointer increment loop: 

```cpp
while (asks_[best_ask_].empty()) 
{
    best_ask_++;
}
```

This search completely stalled the CPU execution pipeline, accumulating about **47 Billion conditional branches** and **4.6 Billion L1 cache misses** under high event loads.

Furthermore, the benchmarking metrics helped me realise that the testing harness was constructing and destroying the `ob::Engine` instance inside the core iteration loops, forcing the operating system to clear and reallocate 48 Megabytes of memory repeatedly. I thus fixed this and implemented a `reset()` function to clear the active array counters and zero out the index boundaries, while preserving the memory.

To completely eliminate the 47 billion branch loop penalty while maintaining the O(1) lookup, I implemented a Hierarchical Bitmask Index. Instead of verifying array slots one by one, the book tracks populated prices using a two tiered bitset composed of 64bit unsigned integers `std::uint64_t words`:

1) An array where each bit represents an individual price tick. A bit value of 1 indicates an active price level while 0 indicates empty space. **(level 1)**

2) An array where each bit tracks an entire 64bit word of level 1. **(level 2)**

When a trade or cancellation empties a price level, the state transition flips the bit off. When searching for the next active price, the engine uses the second level to bypass thousands of empty ticks instantly, allowing the CPU to calculate the precise memory offset of the next active price level in a single clock cycle.

Integrating the bitmask showed performance benefits across all testing scripts: 

- Sparse Market `deep_script.txt`: The engine achieved an execution latency of about **30ns** per event, down from the previous **643ns**. This is a result of conditional branches dropping from **47 Billion to 590 Million** and the L1 data cache misses dropping from **4.6 Billion to 16 Million**. Comparing that to our original `std::map` engine's **656ns** baseline, demonstrates the great improvement. 

- Dense Market `dense_script.txt`: The optimised engine completed matching sweeps in about **26ns** per event compared to the original engine's **90ns** baseline, proving that removing tree layers gives around about a **3x** performance increase when operations are clustered. 

- Basic Workload `script.txt`: Possibly as a surprise, the original engine tracks slightly faster on this script (**~58ns vs ~112ns**) due to the zero indexing abstractions under an artificial, warm L1 footprint.

*Noting that the optimised engine maintains invariant execution times regardless of whether the book contains a dozen orders or millions of rows.*

Optimising for "average execution speed" on clean, isolated loops is basically a vanity metric. 

What dictates production survival is deterministic tail latency under heavy load. A standard library tree + list architecture degrades logarithmically as volume scales. 

When transaction volumes increase drastically, a single `std::map::insert` node allocation will eventually trigger a page fault or heap reorganisation. When this occurs, latency will spike from nanoseconds into microseconds. 

(You can have a look, as I left the first iteration on the original-implementation branch). 

In my updated implementation, I isolated the execution pathway entirely from the kernel. This architecture ensures that order throughput remains a flat, predictable timeline under market pressure.

---

## Folder layout From the root: 

`src/` Library and app source 

`tests/` Googletest suite 

`examples/` Sample scripts

`build/` Build output (not committed)

--- 

Follow these instructions to compile, verify, and run benchmarks against the matching engine on your local machine.

## System Prerequisites

Before configuring the software, ensure your Linux machine has a modern C++17 compliant compiler, CMake, and Git installed. Run the following command to pull all required system dependencies:

```shell
sudo apt-get install build-essential cmake git
```

## Codebase Cloning: 

The codebase relies on GoogleTest as an external git submodule for its unit testing suite. You must clone the repository recursively to automatically initialise and pull down the testing framework components:

```shell
git clone --recursive https://github.com/l1am-farrugia/cppOrderbook.git
```

## Engine Compilation:

This repository utilises CMake to manage optimised release builds. Create a isolation compilation directory, configure CMake to build with high level Release optimisations and compile the binaries:

```shell
mkdir build && cd build

cmake -DCMAKE_BUILD_TYPE=Release ..

make -j$(nproc)
```

Upon a successful build, the following native executable binaries will be generated inside your current `build/` folder:

`ob_tests`: The structural unit testing suite. 

`ob_sim`: The command line trading simulator, replay verifier, and hot path benchmarking tool.

## Running the Unit Testing Suite: 

To ensure that the engine's custom memory pools, intrusive lists, and array tracking boundaries are functioning on your machines architecture, execute the native `GoogleTest` binary:

```shell
./ob_tests
```

## Operating the Engine Simulator: 

The `ob_sim` binary allows you to feed raw trading scripts into the engine. These script files must be text documents containing one instruction per line. The supported command syntax is as follows:

Add Limit Order: `add <OrderId> <Side> <PriceTicks> <Qty>`

`add 5001 buy 1250 50` ← Places a buy limit order with ID 5001 at a price of 1250 ticks for a volume of 50 units. 

Cancel Order: `cancel <OrderId>`

`cancel 5001` ← Locates resting order 5001 and removes it from the queue. 

## Execution:

To run the standard sequential script with console logging enabled use:

```shell
./ob_sim --script ../examples/script.txt
```
 
To run a high volume, 50,000 message clustered script use:

```shell
./ob_sim --bench ../examples/dense_script.txt --iters 100 
```

To run a high volume, widely distributed 200,000 tick script use: 

```
./ob_sim --bench ../examples/deep_script.txt --iters 100
```

I also recommend prefixing the instructions with: 

```shell
sudo perf stat -e L1-dcache-load-misses,cache-misses,branches,branch-misses 
```

In order to gather more metrics on the CPU. 

**note, keep `--iters` <= 100 for the macro scripts because the simulation's isolation reset routine introduces artificial memory bus overhead that distorts the engine's metrics.*

**Thanks for reading :)**
