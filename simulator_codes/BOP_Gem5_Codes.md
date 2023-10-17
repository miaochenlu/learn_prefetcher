# BOP Gem5 Code Reading
# 1. 成员变量

* Recent Request Table
```cpp
std::vector<Addr> rrTable;
```

* Offset List & Score Table
```cpp
/** Structure to save the offset and the score */
typedef std::pair<int16_t, uint8_t> OffsetListEntry;
std::vector<OffsetListEntry> offsetsList;
```

* 一些常量
```cpp
/** Learning phase parameters */
const unsigned int scoreMax;
const unsigned int roundMax;
const unsigned int badScore;
/** Recent requests table parameteres */
const unsigned int rrEntries;
const unsigned int tagMask;
```

# 2. 主要函数
## 2.1 Offset List Init
首先在初始化函数中会设置好offset list，并将score置零

```cpp
const int offsetVal[] = { 1, 2, 3, 4, 5, 6, 8, 9, 10, 12, 15, 16 };
for (int n: offsetVal) {
	offsetsList.push_back(OffsetListEntry(n, 0));
}
```


## 2.2 Best Offset Learning
![](attachments/Pasted%20image%2020230606164008.png)
`bestOffsetLearning(Addr x)`函数会学习best offset
```cpp
void
Xiangshan::bestOffsetLearning(Addr x)
{
    Addr offset_addr = (*offsetsListIterator).first;
    Addr lookup_addr = (x >> lBlkSize) - offset_addr;

    // There was a hit in the RR table, increment the score for this offset
    if (testRR(lookup_addr)) {
        DPRINTF(HWPrefetch, "Address %#lx found in the RR table\n", x);
        (*offsetsListIterator).second++;
        if ((*offsetsListIterator).second > bestScore) {
            bestScore = (*offsetsListIterator).second;
            phaseBestOffset = (*offsetsListIterator).first;
            DPRINTF(HWPrefetch, "New best score is %lu\n", bestScore);
        }
    }

    offsetsListIterator++;

    // All the offsets in the list were visited meaning that a learning
    // phase finished. Check if
    if (offsetsListIterator == offsetsList.end()) {
        offsetsListIterator = offsetsList.begin();
        round++;

        // Check if the best offset must be updated if:
        // (1) One of the scores equals SCORE_MAX
        // (2) The number of rounds equals ROUND_MAX
        if ((bestScore >= scoreMax) || (round == roundMax)) {
            if (bestScore <= badScore) {
                issuePrefetchRequests = false;
            }
            else {
                issuePrefetchRequests = true;
            }
            bestOffset = phaseBestOffset;
            round = 0;
            bestScore = 0;
            phaseBestOffset = 0;
            resetScores();
        }
    }
}
```
主要分为以下几步
* 选定这轮的offset, 这里是取到当前Iterator所在的位置
```cpp
Addr offset_addr = (*offsetsListIterator).first;
```
* 检查request addr的base addr (request addr - offset)是否落在RR Table
	* 如果落在RR Table，则给当前offset加分
```cpp

    // There was a hit in the RR table, increment the score for this offset
    if (testRR(lookup_addr)) {
        DPRINTF(HWPrefetch, "Address %#lx found in the RR table\n", x);
        (*offsetsListIterator).second++;
        if ((*offsetsListIterator).second > bestScore) {
            bestScore = (*offsetsListIterator).second;
            phaseBestOffset = (*offsetsListIterator).first;
            DPRINTF(HWPrefetch, "New best score is %lu\n", bestScore);
        }
    }
    offsetsListIterator++;
```
* 如果所有offset都被测试过一遍了，看是否结束Learning
	* 如果有offset分数大于SCOREMAX或者round达到了ROUNDMAX则结束，并且更新best offset
```cpp
    // All the offsets in the list were visited meaning that a learning
    // phase finished. Check if
    if (offsetsListIterator == offsetsList.end()) {
        offsetsListIterator = offsetsList.begin();
        round++;

        // Check if the best offset must be updated if:
        // (1) One of the scores equals SCORE_MAX
        // (2) The number of rounds equals ROUND_MAX
        if ((bestScore >= scoreMax) || (round == roundMax)) {
            if (bestScore <= badScore) {
                issuePrefetchRequests = false;
            }
            else {
                issuePrefetchRequests = true;
            }
            bestOffset = phaseBestOffset;
            round = 0;
            bestScore = 0;
            phaseBestOffset = 0;
            resetScores();
        }
    }
```

## 2.3 RR Table Update

![](attachments/Pasted%20image%2020230606164025.png)

`notifyFill(const PacketPtr& pkt)`函数会insert entry到RR Table
* 先判断Refill是否是prefetch请求，不是则结束
* 将prefetch addr减去offset的值插入RR Table

```cpp
void
Xiangshan::notifyFill(const PacketPtr& pkt)
{
    // Only insert into the RR right way if it's the pkt is a HWP
    if (!pkt->cmd.isHWPrefetch()) return;

    if (issuePrefetchRequests) {
        Addr inserted = (pkt->getAddr() >> lBlkSize) - bestOffset;
        if (samePage(inserted << lBlkSize, pkt->getAddr())) {
            insertIntoRR(inserted);
        }
    }
}
```

## 2.4 Prefetch
![](attachments/Pasted%20image%2020230606164455.png)
`calculatePrefetch`函数
将addr加上best offset发出prefetch request
```cpp
void
Xiangshan::calculatePrefetch(const PrefetchInfo &pfi,
        std::vector<AddrPriority> &addresses)
{
    Addr addr = pfi.getAddr();

    // Go through the nth offset and update the score, the best score and the
    // current best offset if a better one is found
    bestOffsetLearning(addr);

    // This prefetcher is a degree 1 prefetch, so it will only generate one
    // prefetch at most per access
    if (issuePrefetchRequests) {
        Addr prefetch_addr = addr + (bestOffset << lBlkSize);
        if (samePage(prefetch_addr, addr)) {
            addresses.push_back(AddrPriority(prefetch_addr, 0));
        }
    }
}
```