# SPP + PPF代码阅读

参考[Hermes](https://github.com/CMU-SAFARI/Hermes/blob/main/prefetcher/ppf_dev.cc)中的ChampSim实现
<img src="img\Pasted image 20230328153918.png">

# PPF的一些成员变量


## a. Perceptron Structure 
<img src="img\Pasted image 20230404135238.png">

```cpp
class PERCEPTRON {
public:
    // Perc Weights
    int32_t perc_weights[PERC_ENTRIES][PERC_FEATURES];
    // 每个feature最大值，feature % depth
    int32_t PERC_DEPTH[PERC_FEATURES];

    PERCEPTRON() {
        PERC_DEPTH[0] = 2048; //base_addr;
        PERC_DEPTH[1] = 4096; //cache_line;
        PERC_DEPTH[2] = 4096; //page_addr;
        PERC_DEPTH[3] = 4096; //confidence ^ page_addr;
        PERC_DEPTH[4] = 1024; //curr_sig ^ sig_delta;
        PERC_DEPTH[5] = 4096; //ip_1 ^ ip_2 ^ ip_3;
        PERC_DEPTH[6] = 1024; //ip ^ depth;
        PERC_DEPTH[7] = 2048; //ip ^ sig_delta;
        PERC_DEPTH[8] = 128;  //confidence;

        for (int i = 0; i < PERC_ENTRIES; i++)
            for (int j = 0; j < PERC_FEATURES; j++)
                perc_weights[i][j] = 0;
    }

    void perc_update(uint64_t check_addr, uint64_t ip, uint64_t ip_1, uint64_t ip_2, uint64_t ip_3, int32_t cur_delta, uint32_t last_sig, uint32_t curr_sig, uint32_t confidence, uint32_t depth, bool direction, int32_t perc_sum);
    int32_t perc_predict(uint64_t check_addr, uint64_t ip, uint64_t ip_1, uint64_t ip_2, uint64_t ip_3, int32_t cur_delta, uint32_t last_sig, uint32_t curr_sig, uint32_t confidence, uint32_t depth);
    void get_perc_index(uint64_t base_addr, uint64_t ip, uint64_t ip_1, uint64_t ip_2, uint64_t ip_3, int32_t cur_delta, uint32_t last_sig, uint32_t curr_sig, uint32_t confidence, uint32_t depth, uint64_t perc_set[PERC_FEATURES]);
};

```

## b. Prefetch Filter Structure
PPF主要由两个table构成
* Prefetch Table 
* Reject Table
这两个Table的结构论文里面是这么说的

> The Prefetch table is a 1024-entry, direct mapped structure that contains all metadata required to re-index the perceptron entries for training.
> Ten bits of the addr are used to index into the tables, and another six bits are stored to perform tag matching.

<img src="img\Pasted image 20230404132431.png">
对应到代码中table的结构

```cpp
bool valid[FILTER_SET]; // Consider this as "prefetched"
uint64_t remainder_tag[FILTER_SET];
bool useful[FILTER_SET]; // Consider this as "used"
int32_t perc_sum[FILTER_SET];
uint64_t pc[FILTER_SET];
uint64_t pc_1[FILTER_SET];
uint64_t pc_2[FILTER_SET];
uint64_t pc_3[FILTER_SET];
uint64_t address[FILTER_SET];
uint32_t last_signature[FILTER_SET];
uint32_t cur_signature[FILTER_SET];
int32_t delta[FILTER_SET];
uint32_t confidence[FILTER_SET];
uint32_t la_depth[FILTER_SET];
```

# Process
## a. PPF training & Feedback and Data Retrieval
主要在两个时间点进行PPF training, 一个是demand request来的时候，另一个是cache eviction的时候
### i. On Demand Request
在接收到demand request之后，除了操作ST和PT之外，还需要更新Filter的内容
论文中如此描述
> The addr from the demand request triggering the training is looked up in Prefetch Table and Reject Table.
> 
> == 访问prefetch table更新权重
>
> If the addr is in the prefetch table and marked as valid, this hints the prev prediction was correct and this is a useful prefetch. We compute the sum of the corresponding weights.
>
> If the sum falls below a specific threshold, training occurs and the corresponding weights are adjusted accordingly.
>
>// 访问reject table更新权重  
>Parallel to accessing the prefetch table, the reject table is accessed.
>Before the demand access triggers the next set of prefetches, the reject table is checked for a valid entry.   
>
>A hit means that the corresponding cache block was initially suggested by the underlying prefetcher, but wrongly rejected by the perceptron filter.  
>
>The perceptron filter learns from this and makes use of the corresponding features associated to the original prefetch request, which are stored in the reject table, to index the weights tables and asjut the weights accordingly.

```cpp
void SPP_PPF_dev::invoke_prefetcher(uint64_t ip, uint64_t addr, uint8_t cache_hit, uint8_t type, std::vector<uint64_t> &pref_addr) {
    uint64_t ...
    // Stage 1: Read and update a sig stored in ST
    // last_sig and delta are used to update (sig, delta) correlation in PT
    // curr_sig is used to read prefetch candidates in PT 
    ST.read_and_update_sig(page, page_offset, last_sig, curr_sig, delta);
    // Also check the prefetch filter in parallel to update global accuracy counters 
    FILTER.check(addr, 0, 0, L2C_DEMAND, 0, 0, 0, 0, 0, 0); 
...
```

#### Filter `check`
首先，计算一下addr在两个table中的index和tag

```cpp
uint64_t cache_line = check_addr >> LOG2_BLOCK_SIZE,
	 hash = spp_ppf::get_hash(cache_line);

//MAIN FILTER 计算index和tag
uint64_t quotient = (hash >> REMAINDER_BIT) & ((1 << QUOTIENT_BIT) - 1),
	 remainder = hash % (1 << REMAINDER_BIT);

//REJECT FILTER 计算index和tag
uint64_t quotient_reject = (hash >> REMAINDER_BIT_REJ) & ((1 << QUOTIENT_BIT_REJ) - 1),
	 remainder_reject = hash % (1 << REMAINDER_BIT_REJ);
```

如果是L2 cache demand request 
* 如果Prefetch Table Hit
	* 更新Prefetch Table的权重
* 如果Prefetch Table Miss
	* 如果Reject Table Hit
		* 说明reject不正确，调整reject table的权重

```cpp
case L2C_DEMAND:
if ((remainder_tag[quotient] == remainder) && (useful[quotient] == 0)) {
	useful[quotient] = 1;
	if (valid[quotient]) {
		ghr->pf_useful++; // This cache line was prefetched by SPP and actually used in the program
	}
	if (valid[quotient]) {
		// Prefetch leads to a demand hit
		perc->perc_update(address[quotient], pc[quotient], pc_1[quotient], pc_2[quotient], pc_3[quotient], delta[quotient], last_signature[quotient], cur_signature[quotient], confidence[quotient], la_depth[quotient], 1, perc_sum[quotient]);
	}
}
//If NOT Prefetched
if (!(valid[quotient] && remainder_tag[quotient] == remainder)) {
	// AND If Rejected by Perc
	if (valid_reject[quotient_reject] && remainder_tag_reject[quotient_reject] == remainder_reject) {
		// Not prefetched but could have been a good idea to prefetch
		perc->perc_update(address_reject[quotient_reject], pc_reject[quotient_reject], pc_1_reject[quotient_reject], pc_2_reject[quotient_reject], pc_3_reject[quotient_reject], delta_reject[quotient_reject], last_signature_reject[quotient_reject], cur_signature_reject[quotient_reject], confidence_reject[quotient_reject], la_depth_reject[quotient_reject], 0, perc_sum_reject[quotient_reject]);
		valid_reject[quotient_reject] = 0;
		remainder_tag_reject[quotient_reject] = 0;
	}
}
break;
```

#### 更新权重的函数 `perc_update`

参数中的direction代表了预测是否正确，1代表正确，0代表不正确

```cpp
void PERCEPTRON::perc_update(uint64_t base_addr, uint64_t ip, 
        uint64_t ip_1, uint64_t ip_2, uint64_t ip_3, 
        int32_t cur_delta, uint32_t last_sig, uint32_t curr_sig, 
        uint32_t confidence, uint32_t depth, bool direction, int32_t perc_sum) {
	uint64_t perc_set[PERC_FEATURES];
	// Get the perceptron indexes
	get_perc_index(base_addr, ip, ip_1, ip_2, ip_3, cur_delta, 
		last_sig, curr_sig, confidence, depth, perc_set);
	
	int32_t sum = 0;
	// Restore the sum that led to the prediction
	sum = perc_sum;
	// direction = 1 means the sum was in the correct direction, 
	// 0 means it was in the wrong direction
	if (!direction) {
		// Prediction wrong
		for (int i = 0; i < PERC_FEATURES; i++) {
		// 预测错误，当前预测为预取，但应该不预取，所以调小weight
			if (sum >= knob::ppf_perc_threshold_hi) {
		// Prediction was to prefectch -- so decrement counters
				if (perc_weights[perc_set[i">[i] > -1 * (PERC_COUNTER_MAX + 1) )
					perc_weights[perc_set[i">[i]--;
			}
		// 预测错误，当前预测为不预取，但应该预取，所以调大weight
			if (sum < knob::ppf_perc_threshold_hi) {
		// Prediction was to not prefetch -- so increment counters
				if (perc_weights[perc_set[i">[i] < PERC_COUNTER_MAX)
					perc_weights[perc_set[i">[i]++;
			}
		}
	}
	if (direction && sum > NEG_UPDT_THRESHOLD && sum < POS_UPDT_THRESHOLD) {
		// Prediction correct but sum not 'saturated' enough
		for (int i = 0; i < PERC_FEATURES; i++) {
			if (sum >= knob::ppf_perc_threshold_hi) {
		// Prediction was to prefetch -- so increment counters
				if (perc_weights[perc_set[i">[i] < PERC_COUNTER_MAX)
					perc_weights[perc_set[i">[i]++;
			}
			if (sum < knob::ppf_perc_threshold_hi) {
		// Prediction was to not prefetch -- so decrement counters
				if (perc_weights[perc_set[i">[i] > -1 * (PERC_COUNTER_MAX + 1) )
					perc_weights[perc_set[i">[i]--;
			}
		}
	}
}
```

### ii. On Cache Eviction

论文里是这么说的

> On a cache block eviction, we look up the corresponding addr in the prefetch table. 
> If there is a valid entry with this addr, the filter made a misprediction. 
> Thus, the corresponding features of the prefettch request are used to re-index the tables of weights, and those weights are adjusted according.

```cpp
void SPP_PPF_dev::cache_fill(uint64_t addr, uint32_t set, uint32_t way, uint8_t prefetch, uint64_t evicted_addr) {
#ifdef FILTER_ON
    SPP_DP (cout << endl;);
    FILTER.check(evicted_addr, 0, 0, L2C_EVICT, 0, 0, 0, 0, 0, 0);
#endif
}
```

```cpp
case L2C_EVICT:
// Decrease global pf_useful counter when there is a useless prefetch (prefetched but not used)
if (valid[quotient] && !useful[quotient]) {
	if (ghr->pf_useful) 
		ghr->pf_useful--;

	// Prefetch leads to eviction
	perc->perc_update(address[quotient], pc[quotient], pc_1[quotient], pc_2[quotient], pc_3[quotient], delta[quotient], last_signature[quotient], cur_signature[quotient], confidence[quotient], la_depth[quotient], 0, perc_sum[quotient]);
}
// Reset filter entry
valid[quotient] = 0;
useful[quotient] = 0;
remainder_tag[quotient] = 0;

// Reset reject filter too
valid_reject[quotient_reject] = 0;
remainder_tag_reject[quotient_reject] = 0;
break;
```

## b. PPF Interfencing 

论文里是这么说的 

> Each feature corresponding to a suggested prefetch is used to index a table and all the corresponding weights are summed.
> The sum denotes the confidence value for the suggested prefetch, and is thresholded against two different values: $\tau_{hi}$ and $\tau_{lo}$
> 
> $sum>\tau_{hi}$ -> prefetch into L2
>
> $\tau_{lo}<=sum<=\tau_{hi}$ -> prefetch into LLC

在`read_pattern`中计算出sum, 并且确定prefetch level，放入prefetch queue

```cpp
void PATTERN_TABLE::read_pattern(uint32_t curr_sig, int *delta_q, 
    ...
    if (c_sig[set]) {
        for (uint32_t way = 0; way < PT_WAY; way++) {
            local_conf = (100 * c_delta[set][way]) / c_sig[set];
            pf_conf = depth ? 
                (ghr->global_accuracy * c_delta[set][way] / c_sig[set] * lookahead_conf / 100) 
                : local_conf;

            // 根据PPF获得sum
			int32_t perc_sum = perc->perc_predict(train_addr, curr_ip, 
                    ghr->ip_1, ghr->ip_2, ghr->ip_3, 
                    train_delta + delta[set][way], last_sig, curr_sig, pf_conf, depth);
			bool do_pf = (perc_sum >= knob::ppf_perc_threshold_lo) ? 1 : 0;
			bool fill_l2 = (perc_sum >= knob::ppf_perc_threshold_hi) ? 1 : 0;

			if (fill_l2 && (mshr_occupancy >= mshr_SIZE || pq_occupancy >= pq_SIZE) )
				continue;
			// Now checking against the L2C_MSHR_SIZE
			// Saving some slots in the internal PF queue by checking against do_pf
            if (pf_conf && do_pf && pf_q_tail < 100 ) {
				confidence_q[pf_q_tail] = pf_conf;
            	delta_q[pf_q_tail] = delta[set][way];
				perc_sum_q[pf_q_tail] = perc_sum;
            	// Lookahead path follows the most confident entry
            	if (pf_conf > max_conf) {
            	    lookahead_way = way;
            	    max_conf = pf_conf;
            	}
            	pf_q_tail++;
				found_candidate = true;
            }
			...
    } else confidence_q[pf_q_tail] = 0;
}
```

然后invoke_prefetcher中去发起prefetch请求

```cpp
do_lookahead = 0;
for (uint32_t i = pf_q_head; i < pf_q_tail; i++) {

	uint64_t pf_addr = (base_addr & ~(BLOCK_SIZE - 1)) + (delta_q[i] << LOG2_BLOCK_SIZE);
	int32_t perc_sum   = perc_sum_q[i];

	FILTER_REQUEST fill_level = (perc_sum >= knob::ppf_perc_threshold_hi) ? SPP_L2C_PREFETCH : SPP_LLC_PREFETCH;
	// Prefetch request is in the same physical page	
	if ((addr & ~(PAGE_SIZE - 1)) == (pf_addr & ~(PAGE_SIZE - 1))) {
		
// Filter checks for redundancy and returns FALSE if redundant
// Else it returns TRUE and logs the features for future retrieval 
		if ( num_pf < ceil(((m_parent_cache->PQ.SIZE)/distinct_pages)) ) {                  
			if (FILTER.check(pf_addr, train_addr, curr_ip, fill_level, 
				train_delta + delta_q[i], last_sig, curr_sig, confidence_q[i], perc_sum, (depth-1))) {

				//[DO NOT TOUCH]:   
				// Use addr (not base_addr) to obey the same physical page boundary
				m_parent_cache->prefetch_line(ip, addr, pf_addr, ((fill_level == SPP_L2C_PREFETCH) ? FILL_L2 : FILL_LLC),5); 
				num_pf++;
				
				//FILTER.valid_reject[quotient] = 0;
				if (fill_level == SPP_L2C_PREFETCH) {
					GHR.pf_issued++;
					if (GHR.pf_issued > GLOBAL_COUNTER_MAX) {
						GHR.pf_issued >>= 1;
						GHR.pf_useful >>= 1;
					}
				}
			}
		}   
	}
```

## c. PPF recording 
### i. record reject table 

在`read_pattern`函数中，如果sum小于threshold（即使LLC prefetch, 也记录)，会记录到reject table

```cpp
// Recording Perc negatives
if (pf_conf && pf_q_tail < L2C_MSHR_SIZE && (perc_sum < knob::ppf_perc_threshold_hi) ) {
// Note: Using knob::ppf_perc_threshold_hi as the decising factor for negative case
// Because 'trueness' of a prefetch is decisded based on the feedback from L2C
// So even though LLC prefetches go through, they are treated as false wrt L2C in this case
	uint64_t pf_addr = (base_addr & ~(BLOCK_SIZE - 1)) + (delta[set][way] << LOG2_BLOCK_SIZE);
	
	if ((addr & ~(PAGE_SIZE - 1)) == (pf_addr & ~(PAGE_SIZE - 1))) { // Prefetch request is in the same physical page
		filter->check(pf_addr, train_addr, curr_ip, 
			SPP_PERC_REJECT, train_delta + delta[set][way], 
			last_sig, curr_sig, pf_conf, perc_sum, depth);
	}
}
```

对应到`filter.check`函数，做的是一个记录操作

```cpp
// To see what would have been the prediction given perceptron has rejected the PF
case SPP_PERC_REJECT:
if ((valid[quotient] || useful[quotient]) && remainder_tag[quotient] == remainder) { 
// We want to check if the prefetch would have gone through had perc not rejected
// So even in perc reject case, I'm checking in the accept filter for redundancy
	return false; // False return indicates "Do not prefetch"
} else {
	valid_reject[quotient_reject] = 1;
	remainder_tag_reject[quotient_reject] = remainder_reject;

	// Logging perc features
	address_reject[quotient_reject] = base_addr;
	pc_reject[quotient_reject] = ip;
	pc_1_reject[quotient_reject] = ghr->ip_1;
	pc_2_reject[quotient_reject] = ghr->ip_2;
	pc_3_reject[quotient_reject] = ghr->ip_3;
	delta_reject[quotient_reject] = cur_delta;
	perc_sum_reject[quotient_reject] = sum;
	last_signature_reject[quotient_reject] = last_sig;
	cur_signature_reject[quotient_reject] = curr_sig;
	confidence_reject[quotient_reject] = conf;
	la_depth_reject[quotient_reject] = depth;
}
break;
```

### ii. record prefetch table 

如果可以做L2 prefetch, 则会加入到prefetch table中

```cpp
FILTER_REQUEST fill_level = (perc_sum >= knob::ppf_perc_threshold_hi) ? SPP_L2C_PREFETCH : SPP_LLC_PREFETCH;

// Prefetch request is in the same physical page
if ((addr & ~(PAGE_SIZE - 1)) == (pf_addr & ~(PAGE_SIZE - 1))) {
	// Filter checks for redundancy and returns FALSE if redundant
	// Else it returns TRUE and logs the features for future retrieval 
	if ( num_pf < ceil(((m_parent_cache->PQ.SIZE) / distinct_pages)) ) {                  
		if (FILTER.check(pf_addr, train_addr, curr_ip, fill_level, 
			train_delta + delta_q[i], last_sig, curr_sig, confidence_q[i], perc_sum, (depth-1))) {

			//[DO NOT TOUCH]:   
			// Use addr (not base_addr) to obey the same physical page boundary
			m_parent_cache->prefetch_line(ip, addr, pf_addr, ((fill_level == SPP_L2C_PREFETCH) ? FILL_L2 : FILL_LLC),5); 
                        num_pf++;
```

对应到`filter.check`函数，做的是一个记录操作 

```cpp
case SPP_L2C_PREFETCH:
if ((valid[quotient] || useful[quotient]) && remainder_tag[quotient] == remainder) { 
	return false; // False return indicates "Do not prefetch"
} else {
	valid[quotient] = 1;  // Mark as prefetched
	useful[quotient] = 0; // Reset useful bit
	remainder_tag[quotient] = remainder;

	// Logging perc features
	delta[quotient] = cur_delta;
	pc[quotient] = ip;
	pc_1[quotient] = ghr->ip_1;
	pc_2[quotient] = ghr->ip_2;
	pc_3[quotient] = ghr->ip_3;
	last_signature[quotient] = last_sig; 
	cur_signature[quotient] = curr_sig;
	confidence[quotient] = conf;
	address[quotient] = base_addr; 
	perc_sum[quotient] = sum;
	la_depth[quotient] = depth;
}
break;
```