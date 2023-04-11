---
UID: 20230410152450 
aliases: 成员变量
tags: 
source: 
cssclass: 
created: "2023-04-10 15:24"
updated: "2023-04-11 16:03"
---

# 成员变量
* 每个CPU有一个IP table: trackers_l1 
* 每个CPU有一个Delta Pred Table (也就是论文中的CSPT)
* 每个CPU有一个GHB

```cpp
class IPCP_L1 : public Prefetcher
{
public:
   CACHE *m_parent_cache;
   IP_TABLE_L1 trackers_l1[NUM_CPUS][NUM_IP_TABLE_L1_ENTRIES];
   DELTA_PRED_TABLE DPT_l1[NUM_CPUS][4096];
   uint64_t ghb_l1[NUM_CPUS][NUM_GHB_ENTRIES];
   uint64_t prev_cpu_cycle[NUM_CPUS];
   uint64_t num_misses[NUM_CPUS];
   float mpkc[NUM_CPUS] = {0};
   int spec_nl[NUM_CPUS] = {0};
   ...
```

## 1. IP table的结构

<img src="img\Pasted image 20230411133525.png">

```cpp
class IP_TABLE_L1
{
public:
    uint64_t ip_tag;
    uint64_t last_page;       // last page seen by IP
    uint64_t last_cl_offset;  // last cl offset in the 4KB page
    int64_t last_stride;      // last delta observed
    uint16_t ip_valid;        // Valid IP or not
    int conf;                 // CS conf
    uint16_t signature;       // CPLX signature
    uint16_t str_dir;         // stream direction
    uint16_t str_valid;       // stream valid
    uint16_t str_strength;    // stream strength} 
```

## 2. DELTA_PRED_TABLE
table里面的每个entry存一个delta和一个conf

```cpp
class DELTA_PRED_TABLE
{
public:
    int delta;
    int conf;
};
```

## 3. Region Stream Table 
<img src="img\Pasted image 20230411141820.png">

```cpp
class REGION_STREAM_TABLE {
public:
	uint64_t region_id;		
	uint64_t tentative_dense;	// tentative dense bit
	uint64_t trained_dense;		// trained dense bit
	uint64_t pos_neg_count;		// positive/negative stream counter
	uint64_t dir;				// direction of stream - 1 for +ve and 0 for -ve
	uint64_t lru;				// lru for replacement
	// bit vector to store which lines in the 2KB region have been accessed
	uint8_t line_access[NUM_OF_LINES_IN_REGION];
};
```

# 2. Process
## a. GS
### i. demand access来了之后，检查rstable，是否落在trained page，设置flag

```cpp
//Checking if IP is already classified as a part of the GS class, so that for the new region we will set the tentative (spec_dense) bit.
for(int i = 0; i < NUM_RST_ENTRIES; i++) {
	if(rstable[cpu][i].region_id == ((trackers_l1[cpu][index].last_vpage << 1) 
		| (trackers_l1[cpu][index].last_line_offset >> 5))) {
		if(rstable[cpu][i].trained_dense == 1)
			flag = 1;
		break;
	}
}
```

### ii. train rstable 
#### 找到对应的region
* 首先在rstable中找到对应的region, 如果没有找到，则需要进行替换，这在下面会讲到。现在假设存在对应的region
* 如果region中的cacheline被第一次访问，则设置line_access bit
* 然后是对stream direction的处理
	* 如果stride > 0, pos_neg_count++ 
	* stride < 0, pos_neg_count--
* 如果当前region当前不是dense region 
	* 对line_access中的bit进行计数，看是否超过75%
	* 如果超过了，则设置成dense region
* 如果访问当前region的IP是GS IP，那么设置tentative_dense bit
* 如果是trained_dense或者tentative_dense
	* 计算stream的direction
	* 更新到IP table的stream fields

```cpp
for(c = 0; c < NUM_RST_ENTRIES; c++) {
	if(((curr_page << 1) | (line_offset >> 5)) == rstable[cpu][c].region_id) {
		if(rstable[cpu][c].line_access[line_offset & REGION_OFFSET_MASK] == 0) {
			rstable[cpu][c].line_access[line_offset & REGION_OFFSET_MASK] = 1;
		}

		if(rstable[cpu][c].pos_neg_count >= MAX_POS_NEG_COUNT 
			|| rstable[cpu][c].pos_neg_count <= 0) {
			rstable[cpu][c].pos_neg_count = MAX_POS_NEG_COUNT/2;
		}

		if(stride > 0)
			rstable[cpu][c].pos_neg_count++;
		else
			rstable[cpu][c].pos_neg_count--;

		if(rstable[cpu][c].trained_dense == 0) {
			int count = 0;
			for(int i = 0; i < NUM_OF_LINES_IN_REGION; i++)
				if(rstable[cpu][c].line_access[line_offset & REGION_OFFSET_MASK] == 1)
					count++;

			if(count > 24)	//75% of the cache lines in the region are accessed.       
				rstable[cpu][c].trained_dense = 1;   
		}
		if (flag == 1)
			rstable[cpu][c].tentative_dense = 1;

		if(rstable[cpu][c].tentative_dense == 1 || rstable[cpu][c].trained_dense == 1) {
			if(rstable[cpu][c].pos_neg_count > (MAX_POS_NEG_COUNT/2))
				rstable[cpu][c].dir = 1;	//1 for positive direction
			else
				rstable[cpu][c].dir = 0;	//0 for negative direction
			trackers_l1[cpu][index].str_valid = 1;
			
			trackers_l1[cpu][index].str_dir = rstable[cpu][c].dir;
		}
		else
			trackers_l1[cpu][index].str_valid = 0; 
		break;
	}
}
```

#### 没有找到region
分配一个新的region, 根据lru进行替换
初始化rsentry
根据flat设置tentative_dense bit

```cpp
// curr page has no entry in rstable. Then replace lru.
if (c == NUM_RST_ENTRIES) {
	// check lru
	for (c = 0; c < NUM_RST_ENTRIES; c++) {
		if (rstable[cpu][c].lru == (NUM_RST_ENTRIES - 1))
			break;
	}
	for (int i = 0; i < NUM_RST_ENTRIES; i++) {
		if (rstable[cpu][i].lru < rstable[cpu][c].lru)
			rstable[cpu][i].lru++;
	}
	if (flag == 1)
		rstable[cpu][c].tentative_dense = 1;
	else
		rstable[cpu][c].tentative_dense = 0;

	rstable[cpu][c].region_id = (curr_page << 1) | (line_offset >> 5);
	rstable[cpu][c].trained_dense = 0;
	rstable[cpu][c].pos_neg_count = MAX_POS_NEG_COUNT / 2;
	rstable[cpu][c].dir = 0;
	rstable[cpu][c].lru = 0;
	for (int i = 0; i < NUM_OF_LINES_IN_REGION; i++)
		rstable[cpu][c].line_access[i] = 0;
}
```

### iii. prefetch 
如果stream valid, 根据prefetch degree和stream direction发送prefetch请求
发送前用RR filter进行过滤
如果stream非valid则设置flag为0，开启后面的CS

```cpp
if (trackers_l1[cpu][index].str_valid == 1) { // stream IP
	// for stream, prefetch with twice the usual degree
	if (prefetch_degree[cpu][1] < 3)
		flag = 1;
	meta_counter[cpu][0]++;
	total_count[cpu]++;
	for (int i = 0; i < prefetch_degree[cpu][1]; i++) {
		uint64_t pf_address = 0;

		if (trackers_l1[cpu][index].str_dir == 1) { // +ve stream
			pf_address = (line_addr + i + 1) << LOG2_BLOCK_SIZE;
			metadata = encode_metadata(1, S_TYPE, spec_nl[cpu]); // stride is 1
		} else { // -ve stream
			pf_address = (line_addr - i - 1) << LOG2_BLOCK_SIZE;
			metadata = encode_metadata(-1, S_TYPE, spec_nl[cpu]); // stride is -1
		}

		if (acc[cpu][1] < 75)
			metadata = encode_metadata(0, S_TYPE, spec_nl[cpu]);
		// Check if prefetch address is in same 4 KB page
		if ((pf_address >> LOG2_PAGE_SIZE) != (addr >> LOG2_PAGE_SIZE))
			break;

		trackers_l1[cpu][index].pref_type = S_TYPE;

		int found_in_filter = 0;
		for (int i = 0; i < recent_request_filter.size(); i++) {
			if (recent_request_filter[i] == ((pf_address >> 6) & RR_TAG_MASK)) {
				// Prefetch address is present in RR filter
				found_in_filter = 1;
			}
		}
		// Issue prefetch request only if prefetch address is not present in RR filter
		if (found_in_filter == 0) {
			prefetch_line(ip, addr, pf_address, FILL_L1, metadata);
			// Add to RR filter
			recent_request_filter.push_back((pf_address >> 6) & RR_TAG_MASK);
			if (recent_request_filter.size() > NUM_OF_RR_ENTRIES)
				recent_request_filter.erase(recent_request_filter.begin());
		}
		num_prefs++;
	}
}
else
	flag = 1;
```

## b. CS
### i. train 

```cpp
// update constant stride(CS) confidence
trackers_l1[cpu][index].conf = update_conf(stride, trackers_l1[cpu][index].last_stride, trackers_l1[cpu][index].conf);
// update CS only if confidence is zero
if (trackers_l1[cpu][index].conf == 0)
	trackers_l1[cpu][index].last_stride = stride;
```

对应的update_conf函数

```cpp
/*If the actual stride and predicted stride are equal, then the confidence counter is incremented. */
int update_conf(int stride, int pred_stride, int conf) {
    // use 2-bit saturating counter for confidence
    if (stride == pred_stride) { 
        conf++;
        if (conf > 3)
            conf = 3;
    } else {
        conf--;
        if (conf < 0)
            conf = 0;
    }

    return conf;
}
```

### ii. prefetch 
如果confidence足够的话，根据stride和prefetch degree发送prefetch请求
发送前用RR filter进行过滤

```cpp
if (trackers_l1[cpu][index].conf > 1 && trackers_l1[cpu][index].last_stride != 0 && flag == 1) {
        meta_counter[cpu][1]++;
        total_count[cpu]++;

        if (prefetch_degree[cpu][2] < 2)
            flag = 1;
        else
            flag = 0;

        for (int i = 0; i < prefetch_degree[cpu][2]; i++) {
            uint64_t pf_address = (line_addr + (trackers_l1[cpu][index].last_stride * (i + 1))) << LOG2_BLOCK_SIZE;

            // Check if prefetch address is in same 4 KB page
            if ((pf_address >> LOG2_PAGE_SIZE) != (addr >> LOG2_PAGE_SIZE))
                break;

            trackers_l1[cpu][index].pref_type = CS_TYPE;
            bl_index = hash_bloom(pf_address);
            stats[cpu][CS_TYPE].bl_request[bl_index] = 1;
            if (acc[cpu][2] > 75)
                metadata = encode_metadata(trackers_l1[cpu][index].last_stride, CS_TYPE, spec_nl[cpu]);
            else
                metadata = encode_metadata(0, CS_TYPE, spec_nl[cpu]);

            int found_in_filter = 0;
            for (int i = 0; i < recent_request_filter.size(); i++) {
                if (recent_request_filter[i] == ((pf_address >> 6) & RR_TAG_MASK))
                    // Prefetch address is present in RR filter
                    found_in_filter = 1;
            }
            // Issue prefetch request only if prefetch address is not present in RR filter
            if (found_in_filter == 0) {
                prefetch_line(ip, addr, pf_address, FILL_L1, metadata);
                // Add to RR filter
                recent_request_filter.push_back((pf_address >> 6) & RR_TAG_MASK);
                if (recent_request_filter.size() > NUM_OF_RR_ENTRIES)
                    recent_request_filter.erase(recent_request_filter.begin());
            }
            num_prefs++;
        }
    }
    else
        flag = 1;
```

## c. CPLX

### i. training
* IP对应的signature索引到CSPT
* 如果CSPT中的stride和当前stride一样，则增加confidence, 反之减少
* 更新IP table中的signature

```cpp
last_signature = trackers_l1[cpu][index].signature;
// update complex stride(CPLX) confidence
CSPT_l1[cpu][last_signature].conf = update_conf(stride, CSPT_l1[cpu][last_signature].stride, CSPT_l1[cpu][last_signature].conf);

// update CPLX only if confidence is zero
if (CSPT_l1[cpu][last_signature].conf == 0)
	CSPT_l1[cpu][last_signature].stride = stride;

// calculate and update new signature in IP table
signature = update_sig_l1(last_signature, stride);
trackers_l1[cpu][index].signature = signature;
```

其中update_conf和CS是一样的
update_sig_l1函数如下

```cpp
uint16_t update_sig_l1(uint16_t old_sig, int delta) {
    uint16_t new_sig = 0;
    int sig_delta = 0;

    // 7-bit sign magnitude form, since we need to track deltas from +63 to -63
    sig_delta = (delta < 0) ? (((-1) * delta) + (1 << 6)) : delta;
    new_sig = ((old_sig << 1) ^ sig_delta) & ((1 << NUM_SIG_BITS) - 1);

    return new_sig;
}
```

### ii. prefetch 
跟SPP一样speculate lookahead的prefetch

```cpp
// if conf>=0, continue looking for stride
if (CSPT_l1[cpu][signature].conf >= 0 && CSPT_l1[cpu][signature].stride != 0 && flag == 1) {
	int pref_offset = 0, i = 0; // CPLX IP
	meta_counter[cpu][2]++;
	total_count[cpu]++;

	for (i = 0; i < prefetch_degree[cpu][3] + CPLX_DIST; i++) {
		pref_offset += CSPT_l1[cpu][signature].stride;
		uint64_t pf_address = ((line_addr + pref_offset) << LOG2_BLOCK_SIZE);

		// Check if prefetch address is in same 4 KB page
		if (((pf_address >> LOG2_PAGE_SIZE) != (addr >> LOG2_PAGE_SIZE)) ||
			(CSPT_l1[cpu][signature].conf == -1) ||
			(CSPT_l1[cpu][signature].stride == 0))
			// if new entry in CSPT or stride is zero, break
			break;

		// we are not prefetching at L2 for CPLX type, so encode stride as 0
		trackers_l1[cpu][index].pref_type = CPLX_TYPE;
		metadata = encode_metadata(0, CPLX_TYPE, spec_nl[cpu]);

		// prefetch only when conf>0 for CPLX
		if (CSPT_l1[cpu][signature].conf > 0 && i >= CPLX_DIST) {
			bl_index = hash_bloom(pf_address);
			stats[cpu][CPLX_TYPE].bl_request[bl_index] = 1;
			trackers_l1[cpu][index].pref_type = 3;

			int found_in_filter = 0;
			for (int i = 0; i < recent_request_filter.size(); i++) {
				if (recent_request_filter[i] == ((pf_address >> 6) & RR_TAG_MASK))
					// Prefetch address is present in RR filter
					found_in_filter = 1;
			}
			// Issue prefetch request only if prefetch address is not present in RR filter
			if (found_in_filter == 0) {
				prefetch_line(ip, addr, pf_address, FILL_L1, metadata);
				// Add to RR filter
				recent_request_filter.push_back((pf_address >> 6) & RR_TAG_MASK);
				if (recent_request_filter.size() > NUM_OF_RR_ENTRIES)
					recent_request_filter.erase(recent_request_filter.begin());
			}
			num_prefs++;
		}
		signature = update_sig_l1(signature, CSPT_l1[cpu][signature].stride);
	}
}
```

## d. NL prefetcher 
如果GS/CS/CPLX都没有发送预取，则到NL Prefetcher

```cpp
// if no prefetches are issued till now, speculatively issue a next_line prefetch
if (num_prefs == 0 && spec_nl[cpu] == 1) {
	if (flag_nl[cpu] == 0)
		flag_nl[cpu] = 1;
	else {
		uint64_t pf_address = ((addr >> LOG2_BLOCK_SIZE) + 1) << LOG2_BLOCK_SIZE;
		if ((pf_address >> LOG2_PAGE_SIZE) != (addr >> LOG2_PAGE_SIZE)) {
			// update the IP table entries and return if NL request is not to the same 4 KB page
			trackers_l1[cpu][index].last_line_offset = line_offset;
			trackers_l1[cpu][index].last_vpage = curr_page;
			return;
		}
		bl_index = hash_bloom(pf_address);
		stats[cpu][NL_TYPE].bl_request[bl_index] = 1;
		metadata = encode_metadata(1, NL_TYPE, spec_nl[cpu]);

		int found_in_filter = 0;
		for (int i = 0; i < recent_request_filter.size(); i++){
			if (recent_request_filter[i] == ((pf_address >> 6) & RR_TAG_MASK))
				// Prefetch address is present in RR filter
				found_in_filter = 1;
		}
		// Issue prefetch request only if prefetch address is not present in RR filter
		if (found_in_filter == 0) {
			prefetch_line(ip, addr, pf_address, FILL_L1, metadata);
			// Add to RR filter
			recent_request_filter.push_back((pf_address >> 6) & RR_TAG_MASK);
			if (recent_request_filter.size() > NUM_OF_RR_ENTRIES)
				recent_request_filter.erase(recent_request_filter.begin());
		}
		trackers_l1[cpu][index].pref_type = NL_TYPE;
		meta_counter[cpu][3]++;
		total_count[cpu]++;

		if (acc[cpu][4] < 40)
			flag_nl[cpu] = 0;
	} // NL IP
}
```

注意NL Prefetcher会根据MPKI进行控制

```cpp
// update spec nl bit when num misses crosses certain threshold
if (num_misses[cpu] % 256 == 0 && cache_hit == 0) {
	mpki[cpu] = ((num_misses[cpu] * 1000.0) / (ooo_cpu[cpu].num_retired - ooo_cpu[cpu].warmup_instructions));

	if (mpki[cpu] > spec_nl_threshold)
		spec_nl[cpu] = 0;
	else
		spec_nl[cpu] = 1;
}
```

## 4. Others 
#### prefetch degree throttle 

```cpp
// Updating prefetch degree based on accuracy
for (int i = 0; i < 5; i++) {
	if (pref_filled[cpu][i] % 256 == 0) {
		acc_useful[cpu][i] = acc_useful[cpu][i] / 2.0 + (pref_useful[cpu][i] - acc_useful[cpu][i]) / 2.0;
		acc_filled[cpu][i] = acc_filled[cpu][i] / 2.0 + (pref_filled[cpu][i] - acc_filled[cpu][i]) / 2.0;

		if (acc_filled[cpu][i] != 0)
			acc[cpu][i] = 100.0 * acc_useful[cpu][i] / (acc_filled[cpu][i]);
		else
			acc[cpu][i] = 60;

		if (acc[cpu][i] > 75) {
			prefetch_degree[cpu][i]++;
			if (i == 1) {
				// For GS class, degree is incremented/decremented by 2.
				prefetch_degree[cpu][i]++;
				if (prefetch_degree[cpu][i] > 6)
					prefetch_degree[cpu][i] = 6;
			}
			else if (prefetch_degree[cpu][i] > 3)
				prefetch_degree[cpu][i] = 3;
		}
		else if (acc[cpu][i] < 40) {
			prefetch_degree[cpu][i]--;
			if (i == 1)
				prefetch_degree[cpu][i]--;
			if (prefetch_degree[cpu][i] < 1)
				prefetch_degree[cpu][i] = 1;
		}
	}
}
```