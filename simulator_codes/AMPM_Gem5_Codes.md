# AMPM Gem5 Code Reading

# 1. 成员变量

AMPMPrefetcher里面的主要成员变量是`AccessMapPatternMatching`

`AccessMapPatternMatching`的主要成员变量有

## 1.1. Access Map Table

每个entry对应一个cache line的状态机

```cpp
struct AccessMapEntry : public TaggedEntry {
	/** vector containing the state of the cachelines in this zone */
	std::vector<AccessMapState> states;

	AccessMapEntry(size_t num_entries)
	  : TaggedEntry(), states(num_entries, AM_INIT) {}

	void
	invalidate() override
	{
		TaggedEntry::invalidate();
		for (auto &entry : states) {
			entry = AM_INIT;
		}
	}
};
/** Access map table */
AssociativeSet<AccessMapEntry> accessMapTable;
```

其中每个cache line状态机可能处于如下几种状态

```cpp
/** Data type representing the state of a cacheline in the access map */
enum AccessMapState {
	AM_INIT,
	AM_PREFETCH,
	AM_ACCESS,
	AM_INVALID
};
```

## 1.2. 一些统计指标

```cpp
/**
 * Number of good prefetches
 * - State transitions from PREFETCH to ACCESS
 */
uint64_t numGoodPrefetches;
/**
 * Number of prefetches issued
 * - State transitions from INIT to PREFETCH
 */
uint64_t numTotalPrefetches;
/**
 * Number of raw cache misses
 * - State transitions from INIT or PREFETCH to ACCESS
 */
uint64_t numRawCacheMisses;
/**
 * Number of raw cache hits
 * - State transitions from ACCESS to ACCESS
 */
uint64_t numRawCacheHits;
/** Current degree */
unsigned degree;
/** Current useful degree */
unsigned usefulDegree;
```

对应论文里
<div align="center"><img src="attachments/Pasted%20image%2020230509105240.png" width="500"></div>

## 1.3. 一些常量

大部分是用于prefetcher throttling的

```cpp
/** Cacheline size used by the prefetcher using this object */
const unsigned blkSize;
/** Limit the stride checking to -limitStride/+limitStride */
const unsigned limitStride;
/** Maximum number of prefetch generated */
const unsigned startDegree;
/** Amount of memory covered by a hot zone */
const uint64_t hotZoneSize;
/** A prefetch coverage factor bigger than this is considered high */
const double highCoverageThreshold;
/** A prefetch coverage factor smaller than this is considered low */
const double lowCoverageThreshold;
/** A prefetch accuracy factor bigger than this is considered high */
const double highAccuracyThreshold;
/** A prefetch accuracy factor smaller than this is considered low */
const double lowAccuracyThreshold;
/** A cache hit ratio bigger than this is considered high */
const double highCacheHitThreshold;
/** A cache hit ratio smaller than this is considered low */
const double lowCacheHitThreshold;
/** Cycles in an epoch period */
const Cycles epochCycles;
/** Off chip memory latency to use for the epoch bandwidth calculation */
const Tick offChipMemoryLatency;
```

# 2. 成员函数

## 2.1. get access map entry

获取当前request addr的zone, 以及前后相邻的两个zone

```cpp
// Get the entries of the curent block (am_addr), the previous, and the
// following ones
AccessMapEntry *am_entry_curr = getAccessMapEntry(am_addr, is_secure);
AccessMapEntry *am_entry_prev = (am_addr > 0) ?
	getAccessMapEntry(am_addr-1, is_secure) : nullptr;
AccessMapEntry *am_entry_next = (am_addr < (MaxAddr/hotZoneSize)) ?
	getAccessMapEntry(am_addr+1, is_secure) : nullptr;
assert(am_entry_curr != am_entry_prev);
assert(am_entry_curr != am_entry_next);
assert(am_entry_prev != am_entry_next);
assert(am_entry_curr != nullptr);
```

其中`getAccessMapEntry`函数如下: 找到就返回结果，没有找到就进行替换

```cpp
AccessMapPatternMatching::AccessMapEntry *
AccessMapPatternMatching::getAccessMapEntry(Addr am_addr,
                bool is_secure)
{
    AccessMapEntry *am_entry = accessMapTable.findEntry(am_addr, is_secure);
    if (am_entry != nullptr) {
        accessMapTable.accessEntry(am_entry);
    } else {
        am_entry = accessMapTable.findVictim(am_addr);
        assert(am_entry != nullptr);

        accessMapTable.insertEntry(am_addr, is_secure, am_entry);
    }
    return am_entry;
}
```

## 2.2. set entry state

将当前访问zone `am_entry_curr` 的cache line的状态设置为AM_ACCESS, 更新一些统计信息

```cpp
//Mark the current access as Accessed
setEntryState(*am_entry_curr, current_block, AM_ACCESS);
```

其中`setEntryState`函数如下所示

```cpp
void
AccessMapPatternMatching::setEntryState(AccessMapEntry &entry,
    Addr block, enum AccessMapState state) {
    enum AccessMapState old = entry.states[block];
    entry.states[block] = state;
    //do not update stats when initializing
    if (state == AM_INIT) return;
    switch (old) {
        case AM_INIT:
            if (state == AM_PREFETCH) {
                numTotalPrefetches += 1;
            } else if (state == AM_ACCESS) {
                numRawCacheMisses += 1;
            }
            break;
        case AM_PREFETCH:
            if (state == AM_ACCESS) {
                numGoodPrefetches += 1;
                numRawCacheMisses += 1;
            }
            break;
        case AM_ACCESS:
            if (state == AM_ACCESS) {
                numRawCacheHits += 1;
            }
            break;
        default:
            panic("Impossible path\n");
            break;
    }
}
```

## 2.3. issue prefetch requests

先做了一些处理工作，将3个zone的cache line合并到一个vector里面

```cpp
/**
 * Create a contiguous copy of the 3 entries states.
 * With this, we avoid doing boundaries checking in the loop that looks
 * for prefetch candidates, mark out of range positions with AM_INVALID
 */
std::vector<AccessMapState> states(3 * lines_per_zone);
for (unsigned idx = 0; idx < lines_per_zone; idx += 1) {
	states[idx] =
		am_entry_prev != nullptr ? am_entry_prev->states[idx] : AM_INVALID;
	states[idx + lines_per_zone] = am_entry_curr->states[idx];
	states[idx + 2 * lines_per_zone] =
		am_entry_next != nullptr ? am_entry_next->states[idx] : AM_INVALID;
}
```

然后对可能的stride做了一个遍历

```cpp
for (int stride = 1; stride < max_stride; stride += 1) {
	...
}
```

在loop里面会对+stride的可能性和-stride的可能性做一次检查

下面整个是-stride的一段代码

* 先check当前stride是否符合要求(+stride以及+2stride被访问过或者+stride以及+2stride+1被访问过)
* stride符合要求的情况下，更新map，计算pf addr
  * 如果stride > current_block, 那么current_block-stride会落到prev zone, 计算的时候需要注意一下

```cpp
// Test accessed positive strides
if (checkCandidate(states, states_current_block, stride)) {
	// candidate found, current_block - stride
	Addr pf_addr;
	if (stride > current_block) {
		// The index (current_block - stride) falls in the range of
		// the previous zone (am_entry_prev), adjust the address
		// accordingly
		Addr blk = states_current_block - stride;
		pf_addr = (am_addr - 1) * hotZoneSize + blk * blkSize;
		setEntryState(*am_entry_prev, blk, AM_PREFETCH);
	} else {
		// The index (current_block - stride) falls within
		// am_entry_curr
		Addr blk = current_block - stride;
		pf_addr = am_addr * hotZoneSize + blk * blkSize;
		setEntryState(*am_entry_curr, blk, AM_PREFETCH);
	}
	addresses.push_back(Queued::AddrPriority(pf_addr, 0));
	if (addresses.size() == degree) {
		break;
	}
}
```

其中`checkCandidate`函数如下

> [!NOTE]
> 
> 注意target是current - stride不是current + stride

```cpp
/**
 * Given a target cacheline, this function checks if the cachelines
 * that follow the provided stride have been accessed. If so, the line
 * is considered a good candidate.
 * @param states vector containing the states of three contiguous hot zones
 * @param current target block (cacheline)
 * @param stride access stride to obtain the reference cachelines
 * @return true if current is a prefetch candidate
 */
inline bool checkCandidate(std::vector<AccessMapState> const &states,
					Addr current, int stride) const
{
	enum AccessMapState tgt   = states[current - stride];
	enum AccessMapState s     = states[current + stride];
	enum AccessMapState s2    = states[current + 2 * stride];
	enum AccessMapState s2_p1 = states[current + 2 * stride + 1];
	return (tgt != AM_INVALID &&
			((s == AM_ACCESS && s2 == AM_ACCESS) ||
			(s == AM_ACCESS && s2_p1 == AM_ACCESS)));
}
```

## 2.4. throttling

<div align="center"><img src="attachments/Pasted%20image%2020230509105320.png" width="500"></div>

```cpp
void
AccessMapPatternMatching::processEpochEvent()
{
    schedule(epochEvent, clockEdge(epochCycles));
    double prefetch_accuracy =
        ((double) numGoodPrefetches) / ((double) numTotalPrefetches);
    double prefetch_coverage =
        ((double) numGoodPrefetches) / ((double) numRawCacheMisses);
    double cache_hit_ratio = ((double) numRawCacheHits) /
        ((double) (numRawCacheHits + numRawCacheMisses));
    double num_requests = (double) (numRawCacheMisses - numGoodPrefetches +
        numTotalPrefetches);
    double memory_bandwidth = num_requests * offChipMemoryLatency /
        cyclesToTicks(epochCycles);

    if (prefetch_coverage > highCoverageThreshold &&
        (prefetch_accuracy > highAccuracyThreshold ||
        cache_hit_ratio < lowCacheHitThreshold)) {
        usefulDegree += 1;
    } else if ((prefetch_coverage < lowCoverageThreshold &&
               (prefetch_accuracy < lowAccuracyThreshold ||
                cache_hit_ratio > highCacheHitThreshold)) ||
               (prefetch_accuracy < lowAccuracyThreshold &&
                cache_hit_ratio > highCacheHitThreshold)) {
        usefulDegree -= 1;
    }
    degree = std::min((unsigned) memory_bandwidth, usefulDegree);
    // reset epoch stats
    numGoodPrefetches = 0.0;
    numTotalPrefetches = 0.0;
    numRawCacheMisses = 0.0;
    numRawCacheHits = 0.0;
}
```
