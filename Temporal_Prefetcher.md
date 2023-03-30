# Temporal Pattern
* Definition: a sequence of addresses that favor being accessed together and in the same order (e.g., if we observe {A; B; C; D} then it is likely for {B; C; D} to follow {A} in the future).
* Source: loops; pointer-based structures are traversed in program
* Shortcomings
	* （准确率低）Temporal prefetching techniques exhibit low accuracy as they do not know where streams end.
	* （无法避免compulsory miss）As temporal prefetchers rely on address repetition, they are unable to prevent compulsory misses (unobserved misses) from happening
	* （存储开销高）As temporal prefetchers require to store the correlation between addresses, they usually impose large storage overhead (tens of megabytes) that cannot be accommodated on-the-chip next to the cores. 

<img src="img\Pasted image 20230128110653.png">


---

# Category 1: Table-Based -- Markov Prefetcher
---
# Category 2: GHB-Based 
## STMS (Sampled Temporal Memory Streaming) 

<img src="img\Pasted image 20230128143520.png">

<img src="img\Pasted image 20230203115638.png">

### Structure
* History Table: a circular FIFO buffer
	* 每次cache miss发生，会将数据放到FIFO buffer的末尾
* Index Table: set-associative structure, stores a pointer for every observed miss addr to its last occurrence in the History Table

### Process
Cache miss发生的时候，prefetcher先查找index table, 找到miss addr，获取对应到history table的指针
prefetcher找到history table中的位置，然后开始为后续地址发起prefetch请求

### Comments
* 访问速度慢，metadata traffic很大。History Table和Index Table很大，都是放在memory里面的。
* metadata没有办法cache, 因为他是以FIFO buffer的形式存在的，但是memory访问是随机的
---
## Domino
[Domino Paper](https://readpaper.com/pdf-annotate/note?pdfId=4716970745175998465)

<img src="img\Pasted image 20230222102526.png">

### Structure
* History Table (HT): a circular FIFO buffer --> in DRAM
* Enhanced Index Table (EIT): set-associative structure, stores a pointer for every observed miss addr to its last occurrence in the History Table --> in DRAM
<img src="img\Pasted image 20230222102505.png">
Other Storage elements next to each core
* a buffer to record the sequence of triggering events named **LogMiss** （存放两个miss entry)
* a buffer to keep the prefetched cache blocks named **Prefetch Buffer**
* a buffer that holds the sequence of addresses of an active stream named **PointBuf**
* a buffer named **FetchBuf** (存放EIT的row)

### Process
Triggerring Events: cache misses & prefetch hits

#### Recording

* <font color="#2DC26B">Triggering</font>

Upon a triggering event, the address of the triggering event is appended to LogMiss. (LogMiss has the capacity of two cache blocks.)

* <font color="#2DC26B">Updating HT</font>

When one cache block worth of data is in LogMiss, Domino writes it to the end of the HT in the main memory and statistically updates the pointers of the written miss addresses in the EIT. (不是每一次都会更新EIT的)

* <font color="#2DC26B">Updating EIT</font>

To update the EIT, for one out of every N triggering events (e.g., eight) written into the HT, the corresponding row of the EIT is fetched into FetchBuf. 
If a super-entry for the triggering event is not found in the fetched row, a new super-entry is allocated with the LRU policy. 

In the chosen super-entry, Domino attempts to find an entry for the address following the triggering event. If no match is found, a new entry is allocated with LRU policy. The pointer field of the entry is updated to point to the last row of the HT. 
Finally, Domino updates the LRU stack of entries and super-entries. Once Domino is done with the row, it is written back to the EIT.

#### Replaying (replay first, record second)

When the next triggering event occurs (miss or prefetch hit), Domino searches the super-entry and picks the entry for which the address field matches the triggering event (might not be the most recent entry). 
In case no match is found, the stred aam is discardend Domino uses the triggering event to bring another row from the EIT, and otherwise, Domino creates an active stream using the matched entry. It means that Domino sends a request to read the row of the HT pointed to by the pointer field of the matched entry to be brought into PointBuf. 
Once the sequence of miss addresses from the row of the HT arrives, Domino issues prefetch requests to be appended to the Prefetch Buffer.

**Find New Stream**:

Domino prefetcher attempts to find a new stream and replaces the least-recently-used stream with it (which means discarding the contents of the prefetch buffer and PointBuf related to the replaced stream).
To find a new stream, Domino uses the missed address to fetch a row of the EIT. 
When the row is brought into PointBuf, Domino attempts to find the super-entry associated with the missed address. 
In case a match is not found, nothing will be done, and otherwise, a prefetch will be sent for the address field of the most recent entry in the found super-entry to be brought into the Prefetch Buffer.

### Comments
* Domino优化了STMS。Domino可以使用两个地址匹配GHB。

# Category 3: Irregular Stream Buffer
## A. Simple ISB
[ISB paper](https://readpaper.com/pdf-annotate/note?pdfId=4507173464930148353&noteId=1628787819386526208)

The ISB maps **correlated physical addresses to consecutive addresses in a new address space called thestructural address space**. Thus, for ISB, a temporal stream is a sequence of consecutive structural addresses that can be prefetched using a next line prefetcher.
ISB's organization has two benefits. 
1. It can **combine address correlation with PC-localization**, which helps improve coverage and accuracy; PC-localization is a technique that segregates the global stream into sub-streams for each PC, such that each sub-stream is more predictable. Thus, addresses are considered to be correlated only if they appear consecutively in the PC-localized sub-stream. 
2. **ISB's metadata can be cached** in on-chip metadata caches.

<img src="img\Pasted image 20230203115137.png">
<img src="img\Pasted image 20230128144808.png">

### Structure

<img src="img\Pasted image 20230128154912.png">

#### Training Unit
Input
* load PC
* load address
It maintains the **last observed address** in each PC-localized stream. 
It learns pairs of correlated physical addresses and maps these to consecutive structural addresses.

#### Address Mapping Caches (AMCs)
* The Physical-to-Structural AMC (PS-AMC) stores the mapping from the physical address space to the structural address space; it is indexed by physical addresses.
* The Structural-to-Physical AMC (SP-AMC) stores the inverse mapping as the PS-AMC and is indexed by structural addresses.

#### Stream Predictor
Each entry in the stream predictor stores the starting structural address of the temporal stream, a counter to indicate the length of the observed stream, and a counter to indicate the current prefetch look-ahead distance.

#### TLB Sync
the ISB can cache correlation information for just the TLB resident pages and sync the management of this correlation information with TLB misses.
The mapping from the physical address space to the structural address space is cached onchip only for pages that are resident in the TLB, and the prefetcher updates these caches during long latency TLB misses to effectively hide the latency of accessing off-chip meta-data.

### Process
ISB’s three key functions: training, prediction, and TLB eviction
#### A. Training
The training process assigns consecutive structural addresses to the correlated physical addresses that are observed by the training unit.

Example:
the Training Unit’s last observed address is 0xba1f00, whose structural address is 0x1100. 
When the Training Unit receives the physical address 0xca4b00 in the same localized stream, it performs three steps. 
1. It assigns 0xca4b00 the structural address following 0xba1f00’s structural address, namely 0x1101.
2. It updates the PS-AMC entry indexed by physical address 0xca4b00, and it updates the SP-AMC entry indexed by structural address 0x1101. 
3. It changes the last observed address in the Training Unit to 0xca4b00.

<img src="img\Pasted image 20230128155334.png">

#### B. Prediction
Each L2 cache access becomes a trigger address for the prefetcher, causing the PS-AMC to retrieve the trigger address’ structural address.
<img src="img\Pasted image 20230130133929.png">

### TLB evictions
During a TLB eviction, the ISB writes to DRAM any modified mappings for the evicted page, and it fetches from DRAM the structural mapping for the incoming page.

### Experiments
#### Performance

On a single core, the ISB exhibits an average speedup of **23.1% with 93.7% accuracy**, compared to 9.9% speedup and 64.2% accuracy for an idealized prefetcher that overapproximates the STMS prefetcher.
<img src="img\Pasted image 20230202223449.png">
#### Traffic
8.4% memory traffic overhead due to meta-data accesses
#### Storage
* For the hybrid experiments, we use an 8 KB ISB, because a 32 KB ISB provides only a small speedup improvement. 
* For the non-hybrid experiments, we use a **32 KB ISB**, which contains a 16 KB direct-mapped PS-AMC with 32-byte cache lines, and which uses an 8-way set-associative SP-AMC with 32-byte cache lines. (map all pages in a 128 entry data TLB)
* Offchip: It suffices to reserve for the ISB 8 MB of off-chip storage. By contrast, the GHB-based prefetchers that we simulate require up to 128 MB of off-chip storage for the same workloads. T
#### Hybrid
<img src="img\Pasted image 20230131103149.png">

### Comments
* **Fit Level:** L2 Cache
* **Strength**
	* PC localization & address correlation
	* Cached metadata --> fast access & less traffic
* **Weakness**
	* It fetches large amounts of **useless** metadata (90% of its loaded metadata is never used), which leads to poor metadata cache efficiency (30% hit rate) and high metadata traffic overhead (411% traffic overhead for irregular SPEC 2006 workloads).
	* It does not scale to large page sizes. Because the size of the onchip cache grows with the page size, ISB is infeasible for huge pages (2MB to 1GB in size), which are orders of magnitude larger than the standard 4K page, and which are important for many programs that have large memory footprints.
	* It does not work for modern two-level TLBs. If the metadata cache is synchronized with the L1 TLB, the latency of L2 TLB hits is too short to hide the latency of off-chip metadata requests. If the metadata cache is synchronized with the L2 TLB, the metadata cache would be impractically large—on the order of 200-400KB.


## B. Optimized MISB 
[MISB paper](https://readpaper.com/pdf-annotate/note?pdfId=4716967649033060353&noteId=1628838424436038144)
[ARM explanation](https://community.arm.com/arm-research/b/articles/posts/making-temporal-prefetchers-practical--the-misb-prefetcher)

Instead of piggybacking off of the TLB, MISB uses a simple **metadata prefetcher** to load the metadata cache, and it uses an LRU replacement policy to evict lines from the metadata cache.

### Structure
The Training Unit finds correlated addresses within a PC-localized stream and assigns them consecutive structural addresses.
On-chip mappings are stored in the PS and the SP caches, and on eviction, they are written to the corresponding off-chip PS and SP tables. 
The Bloom filter tracks valid PS mappings and filters invalid off-chip requests for PS mappings.
<img src="img\Pasted image 20230129164754.png">
#### Metadata Cache
* PS cache: physical -> structural
* SP cache: structural -> physical

each logical metadata cache line in MISB's PS and SP caches holds one mapping, and on an eviction, the least recently used mapping is evicted. 

#### Metadata Prefetching
on PS and SP metadata cache misses, MISB gets prefetching benefits from fetching a metadata cache line with 8 mappings.

#### Metadata Filtering
many PS load requests are to physical addresses for which MISB has no mapping since they have never been seen before
To filter these requests, MISB uses a Bloom Filter.
In particular, when a new PS mapping is written to off-chip storage, the mapping is added to a Bloom filter. Future PS misses are sent to memory only if the physical address is found in the Bloom filter. 

### Process

<img src="img\Pasted image 20230129235019.png">

### Experiments

#### Performance
* single-core workloads
	* MISB improves performance by 22.7%, compared to 10.6% for an idealized STMS and 4.5% for a realistic ISB
* 4-core multi-programmed workloads
	* MISB improves performance by 19.9%, compared to 7.5% for idealized STMS
<img src="img\Pasted image 20230202223620.png">
#### Traffic
* single-core workloads
	* MISB also significantly reduces off-chip traffic; for SPEC, MISB's traffic overhead of 70% is roughly one fifth of STMS's (342%) and one sixth of ISB's (411%)
#### Storage
**32KB for the on-chip metadata cache and 17KB for the Bloom filter** (Bloom Filter这么大吗)

<img src="img\Pasted image 20230129235941.png">

<img src="img\Pasted image 20230129235933.png">

### Comments

* Fit Level: L2
* Strength
	* Tt dramatically reduces traffic by improving the utilization of ISB's metadata cache and by decoupling metadata cache management from the TLB
	* MISB uses metadata prefetching to hide the latency of off-chip metadata accesses
	* MISB is the first to filter useless metadata traffic using a Bloom filter
* Weakness

# Triage
[Triage paper](https://readpaper.com/pdf-annotate/note?pdfId=4717976427786403841&noteId=1632874443943569152)

Triage present a temporal prefetcher whose metadata resides entirely on chip. 
The key insights are 
* only a small portion of prefetcher metadata is important
* for most workloads with irregular accesses, the benefits of an effective prefetcher outweigh the marginal benefits of a larger data cache. 
Thus, the Triage prefetcher, identifies important metadata and uses a portion of the LLC to store this metadata, and it dynamically partitions the LLC between data and metadata.

<img src="img\Pasted image 20230131164852.png">


## Process
### I. Training
<img src="img\Pasted image 20230203133100.png">
<img src="img\Pasted image 20230131170247.png">

### II. Prediction
<img src="img\Pasted image 20230131170538.png">

### III. Metadata Replacement

**Hawkeye replacement policy**

### IV. Metadata Partition Updates

每隔50000次metadata access, 重新做一次way partition
如果triage要扩张，那么dirty cache line被flush并且标记为invalid供triage使用
如果triage要缩小，那么triage占用的位置被标记为invalid供llc使用

### Experiments

#### Performance
On single-core systems running SPEC 2006 workloads, Triage outperforms state-of-the-art prefetchers that use only on-chip metadata (23.5% speedup for Triage vs. 5.8% for the Best Offset Prefetcher, 2.8% for SMS). 
Triage achieves 70% of the performance of a stateof-the-art temporal prefetcher that uses off-chip metadata (23.5% for Triage vs. 34.7% for MISB).

<img src="img\Pasted image 20230203133820.png">
<img src="img\Pasted image 20230203134225.png">

#### Storage
Occupy LLC 512KB/1MB/Dynamic allocation

#### Hybrid
<img src="img\Pasted image 20230203133953.png">

<img src="img\Pasted image 20230203133345.png">

### Comments
* Fit Level: L2