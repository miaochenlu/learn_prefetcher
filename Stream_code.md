
# Stream Prefetcher Gem5 Code

# 一些参数

```cpp
static const uint32_t MaxContexts = 64; // Creates per-core stream tables for upto 64 processor cores
uint32_t tableSize;       // Number of entries in a stream table
const bool useMasterId;   // Use the master-id to train the streams
uint32_t degree;          // Determines the number of prefetch reuquests to be issued at a time
uint32_t distance;        // Determines the prefetch distance
```

# 一些定义

* Stream Direction

```cpp
// Direction of stream for each stream entry in the stream table
enum StreamDirection{
        ASCENDING = 1,                      // For example - A, A+1, A+2
        DESCENDING = -1,                    // For example - A, A-1, A-2
        INVALID = 0
};
```

* Stream Status

```cpp
// Status of a stream entry in the stream table.
enum StreamStatus{
	INV       = 0,
	TRAINING  = 1,  // Stream training is not over yet. Once trained will move to MONITOR status
	MONITOR   = 2  // Monitor and Request: Stream entry ready for issuing prefetch requests
};
```

* Stream Table Entry

```cpp  class StreamTableEntry {
class StreamTableEntry {
  public:
	int  LRU_index;
	Addr allocAddr;                     // Address that initiated the stream training
	Addr startAddr;                     // First address of a stream
	Addr endAddr;                       // Last address of a stream
	StreamDirection trainedDirection;   // Direction of trained stream (Ascending or Descending)
	StreamStatus    status;             // Status of the stream entry
	StreamDirection trendDirection[2];  // Stores the last two stream directions of an entry
};
```

# 核心函数

## 1. Monitor状态

* 先判断是否落在该stream的检测区间(startAddr, endAddr), 并且判断stream的方向，ascending还是descending
* 根据prefetch degree发送prefetch requests
* 修改stream检测区间 --> (startAddr, endAddr + degree)或者(startAddr + degree, endAddr + degree)

```cpp
switch (table[i]->status) {
case MONITOR:
	if(table[i]->trainedDirection == ASCENDING) {
		// Ascending order
		if((table[i]->startAddr < blk_addr ) && ( table[i]->endAddr > blk_addr)) {
			// Hit to a stream, which is monitored. Issue prefetch requests based on the degree and the direction
			for (uint8_t d = 1; d <= degree; d++) {
				Addr pf_addr = table[i]->endAddr + blkSize * d;
				addresses.push_back(AddrPriority(pf_addr,0));
				DPRINTF(HWPrefetch, "Queuing prefetch to %#x.\n", pf_addr);
			}
			if((table[i]->endAddr + blkSize * degree) - table[i]->startAddr <= distance) {
				table[i]->endAddr   = table[i]->endAddr + blkSize * degree;
			} else {
				table[i]->startAddr = table[i]->startAddr + blkSize * degree;
				table[i]->endAddr   = table[i]->endAddr   + blkSize * degree;
			}
			break;
		}
	} else if(table[i]->trainedDirection == DESCENDING) {
		// 同上
	} else{
		assert(0);
	}
	break;
```

## 2. Training状态
* 判断blockAddr是否位于这个stream的检测范围
* 一个stream可以存两个trend, 所以先设置两个trend
* 如果两个trend匹配，则转换到Monitor状态；否则reset

```cpp
case TRAINING:
if ((abs(table[i]->allocAddr - blk_addr) <= (distance/2) * blkSize) ){
	// Check whether the address is in +/- of distance
	if(table[i]->trendDirection[0] == INVALID){
		table[i]->trendDirection[0] = (blk_addr - table[i]->allocAddr > 0) ? ASCENDING : DESCENDING;
	} else {
		assert(table[i]->trendDirection[1] == INVALID);
		table[i]->trendDirection[1] = (blk_addr - table[i]->allocAddr > 0) ? ASCENDING : DESCENDING;
		if(table[i]->trendDirection[0] == table[i]->trendDirection[1]) {
			table[i]->trainedDirection = table[i]->trendDirection[0];
			table[i]->startAddr = table[i]->allocAddr;
			if(table[i]->trainedDirection != INVALID){
				// Based on the trainedDirection (+1:Ascending, -1:Descending) update the end address of a stream
				table[i]->endAddr = blk_addr + (table[i]->trainedDirection) * blkSize * degree;
			}
			// Entry is ready for issuing prefetch requests
			table[i]->status = MONITOR;
		} else {
			resetEntry(table[i]);
		}
	}
	break;
}
break;
```

## 3. 没有匹配的Stream
* 如果没有stream匹配的话，就要分配一个新的Stream
* 有invalid的stream就用invalid；没有的话根据LRU替换
* 然后设置allocAddr, 切换到Training状态

```cpp
int entry_id;
if(HIT_index!=tableSize) {  //hit
	entry_id = HIT_index;
} else if (INVALID_index!=tableSize) {
	//Existence of invalid streams
	assert(table[INVALID_index]->status == INV);
	table[INVALID_index]->status = TRAINING;
	table[INVALID_index]->allocAddr = blk_addr;
	entry_id = INVALID_index;
} else {
	//Replace the LRU stream-entry
	assert(table[LRU_index]->status!=INV);
	resetEntry(table[LRU_index]);
	table[LRU_index]->status = TRAINING;
	table[LRU_index]->allocAddr = blk_addr;
	entry_id = LRU_index;
}
```

# References

[gem5-with-DPC-2-prefetcher/stream.hh at master · hfsken/gem5-with-DPC-2-prefetcher · GitHub](https://github.com/hfsken/gem5-with-DPC-2-prefetcher/blob/master/src/mem/cache/prefetch/stream.hh)


