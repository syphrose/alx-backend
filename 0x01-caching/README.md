# Cache replacement policies
In computing, cache replacement policies (also known as cache replacement algorithms or cache algorithms) are optimizing instructions or algorithms which a computer program or hardware-maintained structure can utilize to manage a cache of information. Caching improves performance by keeping recent or often-used data items in memory locations which are faster, or computationally cheaper to access, than normal memory stores. When the cache is full, the algorithm must choose which items to discard to make room for new data.

The average memory reference time is;

T = m x Tm + Th + E

where;
m = miss ratio = 1 - (hit ratio)
Tm =  time to make main-memory access when there is a miss (or, with a multi-level cache, average memory reference time for the next-lower cache)
Th = latency: time to reference the cache (should be the same for hits and misses)
E = secondary effects, such as queuing effects in multiprocessor systems

A cache has two primary figures of merit: latency and hit ratio. A number of secondary factors also affect cache performance.

The hit ratio of a cache describes how often a searched-for item is found. More-efficient replacement policies track more usage information to improve the hit rate for a given cache size. The latency of a cache describes how long after requesting a desired item the cache can return that item when there is a hit. Faster replacement strategies typically track of less usage information—or, with a direct-mapped cache, no information—to reduce the time required to update the information. Each replacement strategy is a compromise between hit rate and latency.

Hit-rate measurements are typically performed on benchmark applications, and the hit ratio varies by application. Video and audio streaming applications often have a hit ratio near zero, because each bit of data in the stream is read once (a compulsory miss), used, and then never read or written again. Many cache algorithms (particularly LRU) allow streaming data to fill the cache, pushing out information which will soon be used again (cache pollution). Other factors may be size, length of time to obtain, and expiration. Depending on cache size, no further caching algorithm to discard items may be needed. Algorithms also maintain cache coherence when several caches are used for the same data, such as multiple database servers updating a shared data file.

# Random replacement (RR)
Random replacement selects an item and discards it to make space when necessary. This algorithm does not require keeping any access history. It has been used in ARM processors due to its simplicity, and it allows efficient stochastic simulation.

## Simple queue-based policies

# First in first out (FIFO)
With this algorithm, the cache behaves like a FIFO queue; it evicts blocks in the order in which they were added, regardless of how often or how many times they were accessed before.

# Last in first out (LIFO) or First in last out (FILO)
The cache behaves like a stack, and unlike a FIFO queue. The cache evicts the block added most recently first, regardless of how often or how many times it was accessed before.

## Simple recency-based policies

# Least recently used (LRU)
Discards least-recently-used items first. This algorithm requires keeping track of what was used and when, which is cumbersome. It requires "age bits" for cache lines, and tracks the least-recently-used cache line based on them. When a cache line is used, the age of the other cache lines changes.

# Time-aware, least-recently used
Time-aware, least-recently-used (TLRU) is a variant of LRU designed for when the contents of a cache have a valid lifetime. The algorithm is suitable for network cache applications such as information-centric networking (ICN), content delivery networks (CDNs) and distributed networks in general. TLRU introduces a term: TTU (time to use), a timestamp of content (or a page) which stipulates the usability time for the content based on its locality and the content publisher. TTU provides more control to a local administrator in regulating network storage.

When content subject to TLRU arrives, a cache node calculates the local TTU based on the TTU assigned by the content publisher. The local TTU value is calculated with a locally-defined function. When the local TTU value is calculated, content replacement is performed on a subset of the total content of the cache node. TLRU ensures that less-popular and short-lived content is replaced with incoming content.

# Most-recently-used (MRU)
Unlike LRU, MRU discards the most-recently-used items first. At the 11th VLDB conference, Chou and DeWitt said: "When a file is being repeatedly scanned in a [looping sequential] reference pattern, MRU is the best replacement algorithm."