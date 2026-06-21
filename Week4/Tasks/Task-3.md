
- **Repository:** [syncthing/syncthing](https://github.com/syncthing/syncthing)
    
- **Pull Request:** [cmd/syncthing/cli: indexDumpSize doesn't need a heap (#8024)](https://github.com/syncthing/syncthing/pull/8024)
## What's Premature?

The `indexDumpSize` function (a CLI debug command that lists database entries ordered by size) originally built a full, custom binary heap to handle sorting. This required:

- Creating a custom `SizedElement` type.
    
- Implementing an `ElementHeap` type that fulfilled Go's 5-method `container/heap` interface (`Len`, `Less`, `Swap`, `Push`, `Pop`).
    
- Writing roughly 25 lines of pure boilerplate code.

The code pushed every key/size pair from a database iterator into this heap, then drained the entire structure just to print the results in sorted order.

> **The Flaw:** A heap earns its keep when you need **incremental access**—such as interleaving `extract-min/max` operations with new inserts, or extracting only the top-$k$ elements out of a massive dataset.
> 
> Here, every element was collected upfront before anything was read, and the entire collection was drained at the end. It was essentially a standard heapsort dressed up in a heap's clothes, adding needless conceptual overhead.

## The Simpler Alternative

Append everything to a plain slice while iterating, then call a standard sort function (`sort.Slice`) exactly once before printing.

This is exactly what the merged PR did. By shifting to a simple slice, the author:

- Deleted the entire `ElementHeap` type.
    
- Removed the `container/heap` import entirely.
    
- Net-shrunk the file by **16 lines** (`+15/−31`).
    
- Reduced computational overhead (as noted by the author, a single standard sort typically requires fewer comparisons and swaps than pushing and popping every single element through a heap structure).
    

## What _Would_ Justify Using a Heap?

A heap would have been the correct architectural choice under either of these conditions:

1. **Top-$K$ Filtering:** If the command only needed to output the _top-$k$ largest sizes_ out of a massive dataset instead of the full list. In that case, a min-heap bound to size $k$ optimizes performance because $O(n \log k)$ beats sorting the entire dataset $O(n \log n)$.
    
2. **Streaming Data:** If the results needed to stream out incrementally while new data was still arriving.
    

Because neither condition applied, this was a textbook one-shot, batch job—collect everything, then print everything. A standard sort handles this perfectly with less code complexity.

> **Takeaway:** This is a classic case of using an incremental priority structure for a job that is actually a one-time full sort. It has the exact same architectural smell as building a segment tree for a dataset that never receives updates.
