**Graph 1 - MySQL** FlameGraph
>[!info] [source](https://www.brendangregg.com/FlameGraphs/cpu-mysql-updated.svg)
Since the task says "In your own words," I will use AI just to fix any grammatical or spelling errors. Now, this flame graph is for MySQL, and I've noticed that `row_search_for_mysql` was called in two different branches in the stack, and it accounts for about 50% of the samples. But since there is no plateau, I think that the bottleneck is not in the function itself, but in the query that calls it. Maybe the wider the row, the worse it gets, I guess. :)

**Graph 2 - DTrace kernel** 
>[!info] [source](https://www.brendangregg.com/FlameGraphs/example-dtrace.svg)

I think that `fop_lookup` is a hotspot because it appears twice at different depths, and I don't see an individual slow function that would be a bottleneck.

**Graph 3 - malloc** 
>[!info] [source](https://www.brendangregg.com/FlameGraphs/malloc_perl1.svg)

This is a memory flame graph, not a CPU one, which means that the width represents bytes allocated, not time.

Based on what I've read, the tower here doesn't necessarily mean a memory leak because after the function (or memory, actually) is done, it is freed. So, we can't really know if there is a leak or a bottleneck here, but I've added it because it's a little bit different from the ones above.