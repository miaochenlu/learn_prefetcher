
# DSPatch ChampSim Code Reading

# 1. 成员变量

几大组成部分

* Page Buffer (PB)
* Signature Prediction Table (SPT)

```cpp
deque<DSPatch_PBEntry*> page_buffer;
DSPatch_SPTEntry **spt;
```

## 1.1. PB

<div align="center"><img src="attachments/Pasted%20image%2020230327113927.png" width="400"></div>

```cpp
class DSPatch_PBEntry {
public:
	uint64_t page; // page number
	uint64_t trigger_pc;
	uint32_t trigger_offset; 
	Bitmap bmp_real; // observed bit pattern
	...
};
```

## 1.2. SPT

<div align="center"><img src="attachments/Pasted%20image%2020230327114133.png" width="350"></div>

```cpp
class DSPatch_SPTEntry {
public:
	uint64_t signature;
	Bitmap bmp_cov;
	Bitmap bmp_acc;
	DSPatch_counter measure_covP, measure_accP;
	DSPatch_counter or_count;
	...
};
```

# 2. 成员函数

入口函数为`invoke_prefetcher`

主要过程

1. Learning memory access patterns
    1. 找到PB entry
    2. 记录新的访问pattern
    3. 没有找到的话需要生成新的PB entry。如果PB entry满了的话要evict到SPT里面
2. Generate Prefetches
    1. 访问SPT
    2. 根据DRAM bandwidth选择AccP, CovP，获得candidate
    3. issue prefetch

```cpp
void DSPatch::invoke_prefetcher(uint64_t pc, uint64_t address, 
	uint8_t cache_hit, uint8_t type, vector<uint64_t> &pref_addr) {
	uint64_t page = address >> dspatch_log2_region_size;
	uint32_t offset = (address >> LOG2_BLOCK_SIZE) 
		& ((1ull << (dspatch_log2_region_size - LOG2_BLOCK_SIZE)) - 1);

	DSPatch_PBEntry *pbentry = NULL;
	pbentry = search_pb(page);

	if(pbentry) pbentry->bmp_real[offset] = true;
	else { /* page buffer miss, prefetch trigger opportunity */
		/* insert the new page buffer entry */
		if(page_buffer.size() >= dspatch_pb_size) {
			pbentry = page_buffer.front();
			page_buffer.pop_front();
			add_to_spt(pbentry);
			delete pbentry;
		}
		pbentry = new DSPatch_PBEntry();
		pbentry->page = page;
		pbentry->trigger_pc = pc;
		pbentry->trigger_offset = offset;
		pbentry->bmp_real[offset] = true;
		page_buffer.push_back(pbentry);
		stats.pb.insert++;

		/* trigger prefetch */
		generate_prefetch(pc, page, offset, address, pref_addr);
		if(dspatch_enable_pref_buffer) {
			buffer_prefetch(pref_addr);
			pref_addr.clear();
		}
	}
	/* slowly inject prefetches at every demand access, if buffer is turned on */
	if(dspatch_enable_pref_buffer)
		issue_prefetch(pref_addr);
}
```

## 2.1. Learning memory access patterns

### i. 找到PB entry

```cpp
DSPatch_PBEntry *pbentry = NULL;
pbentry = search_pb(page);
```

主要是这个`search_pb`函数

就是根据当前的page number找到匹配的PB entry

```cpp
DSPatch_PBEntry* DSPatch::search_pb(uint64_t page) {
	auto it = find_if(page_buffer.begin(), page_buffer.end(), 
		[page](DSPatch_PBEntry *pbentry){return pbentry->page == page;});
	return it != page_buffer.end() ? (*it) : NULL;
}
```

### ii. 记录新的访问pattern

在PB entry的`bmp_real`记录当前offset的访问为true

```cpp
if(pbentry)
	/* record the access */
	pbentry->bmp_real[offset] = true;
```

### iii. 没有找到的话需要生成新的PB entry。如果PB entry满了的话要evict到SPT里面

```cpp
// 如果PB满了，evict到SPT
if(page_buffer.size() >= dspatch_pb_size) {
	pbentry = page_buffer.front();
	page_buffer.pop_front();
	add_to_spt(pbentry);
	delete pbentry;
}
// 设置新的PB entry的内容， 插入PB
pbentry = new DSPatch_PBEntry();
pbentry->page = page;
pbentry->trigger_pc = pc;
pbentry->trigger_offset = offset;
pbentry->bmp_real[offset] = true;
page_buffer.push_back(pbentry);
```

PB满了的话，evict到SPT中，主要是`add_to_spt`函数。这个函数应该是代码的重头戏了

```cpp
void DSPatch::add_to_spt(DSPatch_PBEntry *pbentry)
{
	stats.spt.called++;
	Bitmap bmp_real, bmp_cov, bmp_acc;
	bmp_real = pbentry->bmp_real;
	uint64_t trigger_pc = pbentry->trigger_pc;
	uint32_t trigger_offset = pbentry->trigger_offset;

	uint64_t signature = 
		create_signature(trigger_pc, 0xdeadbeef, trigger_offset);
	uint32_t spt_index = get_spt_index(signature);
	assert(spt_index < dspatch_num_spt_entries);
	DSPatch_SPTEntry *sptentry = spt[spt_index];

	bmp_real = BitmapHelper::rotate_right(bmp_real, trigger_offset, 
		dspatch_num_cachelines_in_region);
	bmp_cov  = BitmapHelper::decompress(sptentry->bmp_cov, 
		dspatch_compression_granularity, 
		dspatch_num_cachelines_in_region);
	bmp_acc  = BitmapHelper::decompress(sptentry->bmp_acc, 
		dspatch_compression_granularity, 
		dspatch_num_cachelines_in_region);

	uint32_t pop_count_bmp_real = BitmapHelper::count_bits_set(bmp_real);
	uint32_t pop_count_bmp_cov  = BitmapHelper::count_bits_set(bmp_cov);
	uint32_t pop_count_bmp_acc  = BitmapHelper::count_bits_set(bmp_acc);
	uint32_t same_count_bmp_cov = BitmapHelper::count_bits_same(bmp_cov, bmp_real);
	uint32_t same_count_bmp_acc = BitmapHelper::count_bits_same(bmp_acc, bmp_real);

	uint32_t cov_bmp_cov = 100 * (float)same_count_bmp_cov / pop_count_bmp_real;
	uint32_t acc_bmp_cov = 100 * (float)same_count_bmp_cov / pop_count_bmp_cov;
	uint32_t cov_bmp_acc = 100 * (float)same_count_bmp_acc / pop_count_bmp_real;
	uint32_t acc_bmp_acc = 100 * (float)same_count_bmp_acc / pop_count_bmp_acc;

	/* Update CovP counters */
	if(BitmapHelper::count_bits_diff(bmp_real, bmp_cov) != 0)
		sptentry->or_count.incr(dspatch_or_count_max);
	if(acc_bmp_cov < dspatch_acc_thr || cov_bmp_cov < dspatch_cov_thr)
		sptentry->measure_covP.incr(dspatch_measure_covP_max);

	/* Update CovP */
	if(sptentry->measure_covP.value() == dspatch_measure_covP_max) {
		if(bw_bucket == 3 || cov_bmp_cov < 50) { /* WARNING: hardcoded values */
			sptentry->bmp_cov = 
				BitmapHelper::compress(bmp_real, 
					dspatch_compression_granularity);
			sptentry->or_count.reset();
			stats.spt.bmp_cov_reset++;
		}
	}
	else
	{
		sptentry->bmp_cov = 
			BitmapHelper::compress(BitmapHelper::bitwise_or(bmp_cov, bmp_real), 
				dspatch_compression_granularity);
	}

	/* Update AccP counter(s) */
	if(acc_bmp_acc < 50) /* WARNING: hardcoded value */
		sptentry->measure_accP.incr();
	else
		sptentry->measure_accP.decr();

	/* Update AccP */
	sptentry->bmp_acc = 
		BitmapHelper::bitwise_and(bmp_real, 
			BitmapHelper::decompress(sptentry->bmp_cov, 
				dspatch_compression_granularity, 
				dspatch_num_cachelines_in_region));
	sptentry->bmp_acc = BitmapHelper::compress(sptentry->bmp_acc,
		dspatch_compression_granularity);
}
```

我们拆解这段代码，主要是一下几个步骤

* 找到SPT entry
* 获取bmp_real, bmp_cov, bmp_acc, 进行一系列计算
* 更新CovP
* 更新AccP

#### a. 找到SPT entry

其实是用pc作为signature的

```cpp
uint64_t signature = create_signature(trigger_pc, 0xdeadbeef, trigger_offset);
uint32_t spt_index = get_spt_index(signature);
```

```cpp
// DSPatch organizes the SPT as a 256-entry tagless direct-mapped structure. 
// A simple folded-XOR hash of the PC is used to index into this structure.
uint32_t DSPatch::get_spt_index(uint64_t signature) {
	uint32_t folded_sig = folded_xor(signature, 2);
	uint32_t hashed_index = get_hash(folded_sig);
	return hashed_index % dspatch_num_spt_entries;
}
```

#### b. 获取bmp_real, bmp_cov, bmp_acc, 进行一系列计算

```cpp
bmp_real = pbentry->bmp_real;
// bmp_real先根据trigger_offset进行anchor
bmp_real = BitmapHelper::rotate_right(bmp_real, trigger_offset, 
	dspatch_num_cachelines_in_region);
// 将bmp_cov, bmp_acc解压出来
bmp_cov  = BitmapHelper::decompress(sptentry->bmp_cov, 
	dspatch_compression_granularity, 
	dspatch_num_cachelines_in_region);
bmp_acc  = BitmapHelper::decompress(sptentry->bmp_acc, 
	dspatch_compression_granularity, 
	dspatch_num_cachelines_in_region);

// 看各个里面set为true的有多少，重合的有多少
uint32_t pop_count_bmp_real = BitmapHelper::count_bits_set(bmp_real);
uint32_t pop_count_bmp_cov  = BitmapHelper::count_bits_set(bmp_cov);
uint32_t pop_count_bmp_acc  = BitmapHelper::count_bits_set(bmp_acc);
uint32_t same_count_bmp_cov = BitmapHelper::count_bits_same(bmp_cov, bmp_real);
uint32_t same_count_bmp_acc = BitmapHelper::count_bits_same(bmp_acc, bmp_real);

// 计算spt entry里面的cov和acc
uint32_t cov_bmp_cov = 100 * (float)same_count_bmp_cov / pop_count_bmp_real;
uint32_t acc_bmp_cov = 100 * (float)same_count_bmp_cov / pop_count_bmp_cov;
uint32_t cov_bmp_acc = 100 * (float)same_count_bmp_acc / pop_count_bmp_real;
uint32_t acc_bmp_acc = 100 * (float)same_count_bmp_acc / pop_count_bmp_acc;
```

#### c. 更新CovP counter

```cpp
/* Update CovP counters */

/** OrCount is incremented every time the OR 
 * operationadds any bits to the predicted bit-pattern.
 */
if(BitmapHelper::count_bits_diff(bmp_real, bmp_cov) != 0)
	sptentry->or_count.incr(dspatch_or_count_max);
/** employ a 2b counter called MeasureCovP 
 * to quantify the goodness of CovP (越大说明CovP越不好)
 * Measure CovP is incremented in two cases: 
 * (1) if the CovP prediction accuracy is less than a threshold value (called AccThr ) 
 * (2) if the prefetch coverage from CoP is less than a thresholdvalue (called CovThr )
*/
if(acc_bmp_cov < dspatch_acc_thr || cov_bmp_cov < dspatch_cov_thr)
	sptentry->measure_covP.incr(dspatch_measure_covP_max);

/* Update CovP */

/**DSPatch resets CovP to the current program bit-pattern when Measure CovP 
 * is saturated and either of the two following conditions are satisfied: 
 * (1) current memory bandwidth utilization is in the highest quartile
 * (2) prefetch coverage is less than 50%
 */
if(sptentry->measure_covP.value() == dspatch_measure_covP_max) {
	if(bw_bucket == 3 || cov_bmp_cov < 50) { /* WARNING: hardcoded values */
		sptentry->bmp_cov = BitmapHelper::compress(bmp_real, 
			dspatch_compression_granularity);
		sptentry->or_count.reset();
		stats.spt.bmp_cov_reset++;
	}
}
else {
	// 将bmp_real encode到bmp_cov, 更新
	sptentry->bmp_cov = BitmapHelper::compress(
		BitmapHelper::bitwise_or(bmp_cov, bmp_real), 
		dspatch_compression_granularity);
}
```

#### d. 更新AccP counter

```cpp
// use a 2b counter called MeasureAccP to quantify the goodness of AccP
/**MeasureAccP is incremented if AccP prediction accuracy is less than 50%, 
 * and is decremented otherwise. 
 */
if(acc_bmp_acc < 50)
	sptentry->measure_accP.incr();
else
	sptentry->measure_accP.decr();

sptentry->bmp_acc = BitmapHelper::bitwise_and(bmp_real, 
	BitmapHelper::decompress(sptentry->bmp_cov, dspatch_compression_granularity, 
		dspatch_num_cachelines_in_region));
sptentry->bmp_acc = BitmapHelper::compress(sptentry->bmp_acc, 
	dspatch_compression_granularity);
```

## 2.2. Generate Prefetches

```cpp
/* trigger prefetch */
generate_prefetch(pc, page, offset, address, pref_addr);
```

generate_prefetch函数的内容

```cpp
void DSPatch::generate_prefetch(uint64_t pc, uint64_t page, uint32_t offset, 
	uint64_t address, vector<uint64_t> &pref_addr) {
	Bitmap bmp_cov, bmp_acc, bmp_pred;
	uint64_t signature = 0xdeadbeef;
	DSPatch_pref_candidate candidate = DSPatch_pref_candidate::PAT_NONE;
	DSPatch_SPTEntry *sptentry = NULL;

	stats.gen_pref.called++;
	signature = create_signature(pc, page, offset);
	uint32_t spt_index = get_spt_index(signature);
	assert(spt_index < dspatch_num_spt_entries);
	
	sptentry = spt[spt_index];
	candidate = select_bitmap(sptentry, bmp_pred);

	/* decompress and rotate back the bitmap */
	bmp_pred = BitmapHelper::decompress(bmp_pred, 
		dspatch_compression_granularity, 
		dspatch_num_cachelines_in_region);
	bmp_pred = BitmapHelper::rotate_left(bmp_pred, offset, 
		dspatch_num_cachelines_in_region);

	/* Throttling predictions incase of predicting with bmp_acc and b/w is high */
	if(bw_bucket >= dspatch_pred_throttle_bw_thr 
		&& candidate == DSPatch_pref_candidate::PAT_ACCP)
		bmp_pred.reset();
	
	/* generate prefetch requests */	
	for(uint32_t index = 0; index < dspatch_num_cachelines_in_region; ++index) {
		if(bmp_pred[index] && index != offset) {
			uint64_t addr = (page << dspatch_log2_region_size) 
				+ (index << LOG2_BLOCK_SIZE);
			pref_addr.push_back(addr);
		}
	}
}
```

### i. 访问SPT

```cpp
signature = create_signature(pc, page, offset);
uint32_t spt_index = get_spt_index(signature);
```

### ii. 根据DRAM bandwidth选择AccP, CovP，获得candidate

```cpp
sptentry = spt[spt_index];
candidate = select_bitmap(sptentry, bmp_pred);
```

主要是`select_bitmap`这个函数, `select_bitmap` 函数里面主要是`dyn_selection`函数，对应了论文中的选择过程。最后获得的bit pattern以bmp_pred返回

<div align="center"><img src="attachments/Pasted%20image%2020230327160745.png" width="500"></div>

```cpp
DSPatch_pref_candidate DSPatch::dyn_selection(DSPatch_SPTEntry *sptentry, 
	Bitmap &bmp_selected) {
	DSPatch_pref_candidate candidate = DSPatch_pref_candidate::PAT_NONE;

	if(bw_bucket == 3) {
		if(sptentry->measure_accP.value() == dspatch_measure_accP_max) {
			/* no prefetch */
			bmp_selected.reset();
			candidate = DSPatch_pref_candidate::PAT_NONE;
		} else {
			/* Prefetch with accP */
			bmp_selected = sptentry->bmp_acc;
			candidate = DSPatch_pref_candidate::PAT_ACCP;
		}
	} else if(bw_bucket == 2) {
		if(sptentry->measure_covP.value() == dspatch_measure_covP_max) {
			/* Prefetch with accP */
			bmp_selected = sptentry->bmp_acc;
			candidate = DSPatch_pref_candidate::PAT_ACCP;
		} else {
			/* Prefetch with covP */
			bmp_selected = sptentry->bmp_cov;
			candidate = DSPatch_pref_candidate::PAT_COVP;
		}
	} else {
		/* Prefetch with covP */
		bmp_selected = sptentry->bmp_cov;
		candidate = DSPatch_pref_candidate::PAT_COVP;
	}
	return candidate;
}
```

### iii. issue prefetch

```cpp
// 将bmp_pred解压缩，anchor回来
bmp_pred = BitmapHelper::decompress(bmp_pred, 
	dspatch_compression_granularity, 
	dspatch_num_cachelines_in_region);
bmp_pred = BitmapHelper::rotate_left(bmp_pred, 
	offset, dspatch_num_cachelines_in_region);

/* Throttling predictions incase of predicting with bmp_acc and b/w is high */
if(bw_bucket >= dspatch_pred_throttle_bw_thr 
	&& candidate == DSPatch_pref_candidate::PAT_ACCP) {
	bmp_pred.reset();
}

/* generate prefetch requests */
for(uint32_t index = 0; index < dspatch_num_cachelines_in_region; ++index) {
	if(bmp_pred[index] && index != offset){
		Addr addr = (page << dspatch_log2_region_size) + (index << 6);
		addresses.push_back(AddrPriority(addr, 0));
	}
}
```
