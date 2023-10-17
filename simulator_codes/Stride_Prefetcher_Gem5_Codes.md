# Stride Prefetcher Gem5 Code Reading

# 1. 成员变量

## 1.1. StrideEntryTable

Stride Prefetcher的主要构成是PCTable，每个requestor对应一个PCTable

而PCTable由StrideEntry构成

```cpp
struct StrideEntry : public TaggedEntry {
	StrideEntry(int init_confidence);

	void invalidate() override;

	Addr lastAddr;
	int stride;
	int confidence;
};
typedef AssociativeSet<StrideEntry> PCTable;
std::unordered_map<int, PCTable> pcTables;
```

PCTable的索引方式

<div align="center"><img src="attachments/Pasted%20image%2020230425104319.png" width="500"></div>

```cpp
uint32_t
StridePrefetcherHashedSetAssociative::extractSet(const Addr pc) const
{
    const Addr hash1 = pc >> 1;
    const Addr hash2 = hash1 >> tagShift;
    return (hash1 ^ hash2) & setMask;
}

Addr
StridePrefetcherHashedSetAssociative::extractTag(const Addr addr) const
{
    return addr;
}
```

# 2. 成员函数

* 找到PC table并且根据pc索引到strideEntry

```cpp
```cpp
// Get corresponding pc table
PCTable* pcTable = findTable(requestor_id);

// Search for entry in the pc table
StrideEntry *entry = pcTable->findEntry(pc, is_secure);
```

* 如果strideEntry hit
	1. 更新confidence
	2. 更新stride
	3. 根据prefetch degree issue requests

```cpp
if (entry != nullptr) {
	pcTable->accessEntry(entry);

	// Hit in table
	int new_stride = line_addr - entry->lastAddr;
	bool stride_match = (new_stride == entry->stride);

	// Adjust confidence for stride entry
	if (stride_match && new_stride != 0) {
		if (entry->confidence < threshConf) {
			entry->confidence++;
		}
	} else {
		if (entry->confidence == 0) {
			entry->stride = new_stride;
		}
		else {
			entry->confidence--;
		}
	}

	entry->lastAddr = line_addr;

	// Abort prefetch generation if below confidence threshold
	if (entry->confidence < threshConf) {
		return;
	}

	// Generate up to degree prefetches
	for (int d = 1; d <= currentDegree; d++) {
		// Round strides up to atleast 1 cacheline
		int prefetch_stride = new_stride;

		Addr new_addr = (line_addr + prefetch_stride * d) << lBlkSize;
		if (samePage(pf_addr, new_addr))
			addresses.push_back(AddrPriority(new_addr, 0));
	}
}
```

* 如果strideEntry miss就进行替换

```cpp
else {
	// Miss in table
	StrideEntry* entry = pcTable->findVictim(pc);

	// Insert new entry's data
	entry->lastAddr = line_addr;
	pcTable->insertEntry(pc, is_secure, entry);
}
```

# 3. Throttling

根据prefetch accuracy调节degree

```cpp
void
Stride::notifyFill(const PacketPtr& pkt)
{
    // Only insert into the RR right way if it's the pkt is a HWP
    if (!pkt->cmd.isHWPrefetch()) return;

    filled++;
    if (filled >= adjustInterval) {
        double accuracy = (double) usefulPrefetches / (double) issuedPrefetches;
        if (accuracy > 0.5 && currentDegree < maxDegree) {
            currentDegree++;
        }
        else if (accuracy < 0.2 && currentDegree > 1) {
            currentDegree--;
        }
        filled = 0;
    }
}
```