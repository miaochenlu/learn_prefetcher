# MLOP ChampSim Code Reading

# 1. 成员变量
* Access Map Table
* Offset Score Table
* prefetch level vector

## 1.1 Offset Score Table
Offset Score Table记录了offset对应的得分。
MLOP会为每一个prefetch level记录一个offset score table
```cpp
vector<vector<int>> offset_scores;
offset_scores = vector<vector<int>>(PF_DEGREE, vector<int>(NUM_OFFSETS, 0));
```



## 1.1 Access Map Table
Access Map Table中的每一个entry对应一块memory region (比如说2KB的region)。
entry中记录region中每个cache line的访问情况(一个状态机，这里对应的是`MLOP_State`)
此外，还有一个`prefetch_map`

`prefetch_map`记录了zone里面的某个offset对应的cache line被prefetche

`hist_queue` 以queue的形式存储当前region最近的访问历史(存offset)
```cpp
enum MLOP_State { INIT = 0, ACCESS = 1, PREFTCH = 2 };

class AccessMapData {
  public:
    /* block states are represented with a `MLOP_State` and an `int` in this software implementation but
     * in a hardware implementation, they'd be represented with only 2 bits. */
    vector<MLOP_State> access_map;
    vector<int> prefetch_map;

    deque<int> hist_queue;
};
```


# 2. 整体逻辑

## 2.1 Offset Learning

![](attachments/Pasted%20image%2020230719182601.png)
在每一个prefetch level都对所有offset进行一次学习
假设目前在prefetch level 1,
首先对access_map做一次unmark, 把最后一次access的位置设置为INIT
然后，因为

```cpp
	/* unmark latest access to increase prediction depth */
	if (d != 0) {
		int idx = queue[d - 1];
		// assert(0 <= idx && idx < this->blocks_in_zone);
		access_map[idx] = MLOP_State::INIT;
	}
```

然后
![](attachments/Pasted%20image%2020230720093126.png)
```cpp
	for (uint32_t i = 0; i < this->blocks_in_zone; i += 1) {
		if (access_map[i] == MLOP_State::ACCESS) {
			int offset = zone_offset - i;
			if (offset >= MIN_OFFSET && offset <= MAX_OFFSET && offset != 0) {
				this->offset_scores[d][ORIGIN + offset] += 1;
			}
		}
	}
```


```cpp
this->update_cnt += 1;
const deque<int> &queue = entry->data.hist_queue;
for (int d = 0; d <= (int)queue.size(); d += 1) {
	/* unmark latest access to increase prediction depth */
	if (d != 0) {
		int idx = queue[d - 1];
		// assert(0 <= idx && idx < this->blocks_in_zone);
		access_map[idx] = MLOP_State::INIT;
	}
	for (uint32_t i = 0; i < this->blocks_in_zone; i += 1) {
		if (access_map[i] == MLOP_State::ACCESS) {
			int offset = zone_offset - i;
			if (offset >= MIN_OFFSET && offset <= MAX_OFFSET && offset != 0) {
				this->offset_scores[d][ORIGIN + offset] += 1;
			}
		}
	}
}
```

## 2.1 Mark
mark current access

```cpp
void MLOP::mark(uint64_t block_number, MLOP_State state, int fill_level) {
	this->access_map_table->set_state(block_number, state, fill_level);
}
```

`state_set`函数的主要工作
* 在AMT中找到对应zone的entry 
* 更新access_map的状态，将entry中对应位置的状态设置成ACCESS
* 如果new state是ACCESS， 则将当前位于zone中的pos记录到hist_queue中
* 更新prefetch_map的状态，将当前pos的记录为new_fill_level


## prefetch

两层过滤逻辑
* 如果prefetch line处于ACCESS状态，就不需要prefetch了
* 如果之前prefetch到了更靠近CPU的cache level, 那么由于inclusive特性也不需要prefetch了
```cpp
for (uint32_t d = 0; d < PF_DEGREE; d += 1) {
	for (auto &cur_pf_offset : this->pf_offset[d]) {
		int offset_to_prefetch = zone_offset + cur_pf_offset;

		/* use `access_map` to filter prefetches */
		if (access_map[offset_to_prefetch] == MLOP_State::ACCESS)
			continue;
		if (access_map[offset_to_prefetch] == MLOP_State::PREFTCH &&
				prefetch_map[offset_to_prefetch] <= this->pf_level[d])
			continue;

		if (this->is_inside_zone(offset_to_prefetch) && cache->PQ.occupancy < cache->PQ.SIZE &&
				cache->PQ.occupancy + cache->MSHR.occupancy < cache->MSHR.SIZE - 1) {
			uint64_t pf_block_number = block_number + cur_pf_offset;
			uint64_t base_addr = block_number << LOG2_BLOCK_SIZE;
			uint64_t pf_addr = pf_block_number << LOG2_BLOCK_SIZE;
			cache->prefetch_line(0, base_addr, pf_addr, this->pf_level[d], 0);
			// assert(ok == 1);
			this->mark(pf_block_number, MLOP_State::PREFTCH, this->pf_level[d]);
			pf_issued += 1;
		}
	}
}
```