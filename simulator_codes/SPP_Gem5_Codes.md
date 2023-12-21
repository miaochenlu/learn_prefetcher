# SPP Gem5 Code Reading

# 1. 成员变量

```cpp
/** Signature table */
AssociativeSet<SignatureEntry> signatureTable;
/** Pattern table */
AssociativeSet<PatternEntry> patternTable;
/** Global History Register */
AssociativeSet<GlobalHistoryEntry> globalHistoryRegister;

// 比较重要的threshold
/** Minimum confidence to issue a prefetch */
const double prefetchConfidenceThreshold;
/** Minimum confidence to keep navigating lookahead entries */
const double lookaheadConfidenceThreshold;
```

Gem5中的prefetch filter不需要额外实现
接下来展开说明signatureTable/patternTable/GHR

## 1.1. Signature Table

用page number作为tag进行索引
记录Last offset和signature

<div align="center"><img src="attachments/Pasted%20image%2020230322095800.png" width="150"></div>

```cpp
    /** Signature entry data type */
    struct SignatureEntry : public TaggedEntry
    {
        /** Path signature */
        signature_t signature;
        /** Last accessed block within a page */
        stride_t lastBlock;
        SignatureEntry() : signature(0), lastBlock(0)
        {}
    };
    /** Signature table */
    AssociativeSet<SignatureEntry> signatureTable;
```

## 1.2. pattern Table

以signature为tag做索引

每个signature对应的entry会对应多个delta entry

<div align="center"><img src="attachments/Pasted%20image%2020230322095826.png" width="150"></div>

```cpp
/** Pattern entry data type, a set of stride and counter entries */
struct PatternEntry : public TaggedEntry
{
	/** group of stides */
	std::vector<PatternStrideEntry> strideEntries;
	/** use counter, used by SPPv2 */
	SatCounter8 counter;
	...

	/**
	 * Returns the entry with the desired stride
	 * @param stride the stride to find
	 * @result a pointer to the entry, if the stride was found, or nullptr,
	 *         if the stride was not found
	 */
	PatternStrideEntry *findStride(stride_t stride)
	{
		PatternStrideEntry *found_entry = nullptr;
		for (auto &entry : strideEntries) {
			if (entry.stride == stride) {
				found_entry = &entry;
				break;
			}
		}
		return found_entry;
	}

	/**
	 * Gets the entry with the provided stride, if there is no entry with
	 * the associated stride, it replaces one of them.
	 * @param stride the stride to find
	 * @result reference to the selected entry
	 */
	PatternStrideEntry &getStrideEntry(stride_t stride);
};
/** Pattern table */
AssociativeSet<PatternEntry> patternTable;
```

## 1.3. Global History Register

```cpp
/** Global History Register entry datatype */
struct GlobalHistoryEntry : public TaggedEntry
{
	signature_t signature;
	double confidence;
	stride_t lastBlock;
	stride_t delta;
	GlobalHistoryEntry() : signature(0), confidence(0.0), lastBlock(0),
						   delta(0) {}
};

/** Global History Register */
AssociativeSet<GlobalHistoryEntry> globalHistoryRegister;
```

# 2. 成员函数

核心都在`calculatePrefetch`函数

这个函数主要分成几个阶段

1. Learning Memory Access Patterns
   1. 获取ST entry
   2. 更新PT entry
   3. 更新ST entry
2. Path Confidence-based Prefetching
   1. 根据新的signature获取PT entry
   2. 根据prefetch confidence issue prefetch
   3. 根据lookahead 加深预取
   4. 处理cross page boundary的问题

```cpp
void
SignaturePath::calculatePrefetch(const PrefetchInfo &pfi,
                                 std::vector<AddrPriority> &addresses)
{
    Addr request_addr = pfi.getAddr();
    Addr ppn = request_addr / pageBytes;
    stride_t current_block = (request_addr % pageBytes) / blkSize;
    stride_t stride;
    bool is_secure = pfi.isSecure();
    double initial_confidence = 1.0;

    // Get the SignatureEntry of this page to:
    // - compute the current stride
    // - obtain the current signature of accesses
    bool miss;
    SignatureEntry &signature_entry = getSignatureEntry(ppn, is_secure,
            current_block, miss, stride, initial_confidence);

    if (miss) {
        // No history for this page, can't continue
        return;
    }

    if (stride == 0) {
        // Can't continue with a stride 0
        return;
    }

    // Update the confidence of the current signature
    updatePatternTable(signature_entry.signature, stride);

    // Update the current SignatureEntry signature
    signature_entry.signature =
        updateSignature(signature_entry.signature, stride);

    signature_t current_signature = signature_entry.signature;
    double current_confidence = initial_confidence;
    stride_t current_stride = signature_entry.lastBlock;

    // Look for prefetch candidates while the current path confidence is
    // high enough
    while (current_confidence > lookaheadConfidenceThreshold) {
        // With the updated signature, attempt to generate prefetches
        // - search the PatternTable and select all entries with enough
        //   confidence, these are prefetch candidates
        // - select the entry with the highest counter as the "lookahead"
        PatternEntry *current_pattern_entry =
            patternTable.findEntry(current_signature, false);
        PatternStrideEntry const *lookahead = nullptr;
        if (current_pattern_entry != nullptr) {
            unsigned long max_counter = 0;
            for (auto const &entry : current_pattern_entry->strideEntries) {
                //select the entry with the maximum counter value as lookahead
                if (max_counter < entry.counter) {
                    max_counter = entry.counter;
                    lookahead = &entry;
                }
                double prefetch_confidence =
                    calculatePrefetchConfidence(*current_pattern_entry, entry);

                if (prefetch_confidence >= prefetchConfidenceThreshold) {
                    assert(entry.stride != 0);
                    //prefetch candidate
                    addPrefetch(ppn, current_stride, entry.stride,
                                current_confidence, current_signature,
                                is_secure, addresses);
                }
            }
        }

        if (lookahead != nullptr) {
            current_confidence *= calculateLookaheadConfidence(
                    *current_pattern_entry, *lookahead);
            current_signature =
                updateSignature(current_signature, lookahead->stride);
            current_stride += lookahead->stride;
        } else {
            current_confidence = 0.0;
        }
    }

    auxiliaryPrefetcher(ppn, current_block, is_secure, addresses);
}
```

## 2.1. Learning Memory Access Patterns

### a. 获取ST entry

```cpp
SignaturePath::SignatureEntry &
SignaturePath::getSignatureEntry(Addr ppn, bool is_secure,
        stride_t block, bool &miss, stride_t &stride,
        double &initial_confidence)
{
// 根据ppn查找ST entry
    SignatureEntry* signature_entry = signatureTable.findEntry(ppn, is_secure);
    if (signature_entry != nullptr) {
    //找到了
        signatureTable.accessEntry(signature_entry);
        miss = false;
        stride = block - signature_entry->lastBlock;
    } else {
    //没找到就需要新建一个ST entry, 进行replacement
        signature_entry = signatureTable.findVictim(ppn);
        assert(signature_entry != nullptr);

        // Sets signature_entry->signature, initial_confidence, and stride
        handleSignatureTableMiss(block, signature_entry->signature,
            initial_confidence, stride);

        signatureTable.insertEntry(ppn, is_secure, signature_entry);
        miss = true;
    }
    // 返回ST entry
    signature_entry->lastBlock = block;
    return *signature_entry;
}
```

其中有一个`handleSignatureTableMiss`需要注意一些，这里会查找GHR来看有没有cross page boudary的signature可以复用

```cpp
void
SignaturePathV2::handleSignatureTableMiss(stride_t current_block,
    signature_t &new_signature, double &new_conf, stride_t &new_stride) {
    bool found = false;
    // This should return all entries of the GHR, since it is a fully
    // associative table
    std::vector<GlobalHistoryEntry *> all_ghr_entries =
             globalHistoryRegister.getPossibleEntries(0 /* any value works */);
	// 查找GHR
    for (auto gh_entry : all_ghr_entries) {
        if (gh_entry->lastBlock + gh_entry->delta == current_block) {
            new_signature = gh_entry->signature;
            // SPP会将initial conf设置为GHR里面存储的path conf
            new_conf = gh_entry->confidence;
            new_stride = gh_entry->delta;
            found = true;
            globalHistoryRegister.accessEntry(gh_entry);
            break;
        }
    }
    if (!found) {
        new_signature = current_block;
        new_conf = 1.0;
        new_stride = current_block;
    }
}
```

### b. 更新PT entry

```cpp
void
SignaturePath::updatePatternTable(Addr signature, stride_t stride)
{
    assert(stride != 0);
    // The pattern table is indexed by signatures
    PatternEntry &p_entry = getPatternEntry(signature);
    PatternStrideEntry &ps_entry = p_entry.getStrideEntry(stride);
    increasePatternEntryCounter(p_entry, ps_entry);
}
```

首先要找到PT entry

```cpp
SignaturePath::PatternEntry &
SignaturePath::getPatternEntry(Addr signature) {
    PatternEntry* pattern_entry = patternTable.findEntry(signature, false);
    if (pattern_entry != nullptr) {
        // Signature found
        patternTable.accessEntry(pattern_entry);
    } else {
        // Signature not found
        pattern_entry = patternTable.findVictim(signature);
        assert(pattern_entry != nullptr);

        patternTable.insertEntry(signature, false, pattern_entry);
    }
    return *pattern_entry;
}
```

找到后主要是increase PT entry counter, 来看这个函数

和论文一样，给pattern entry的counter + 1, 给delta对应的counter + 1

如果counter饱和了，就都除以2

```cpp
void
SignaturePathV2::increasePatternEntryCounter(
        PatternEntry &pattern_entry, PatternStrideEntry &pstride_entry) {
    if (pattern_entry.counter.isSaturated()) {
        pattern_entry.counter >>= 1;
        for (auto &entry : pattern_entry.strideEntries) {
            entry.counter >>= 1;
        }
    }
    if (pstride_entry.counter.isSaturated()) {
        pattern_entry.counter >>= 1;
        for (auto &entry : pattern_entry.strideEntries) {
            entry.counter >>= 1;
        }
    }
    pattern_entry.counter++;
    pstride_entry.counter++;
}
```

### c. 更新ST entry

```cpp
/**
 * Generates a new signature from an existing one and a new stride
 * @param sig current signature
 * @param str stride to add to the new signature
 * @result the new signature
 */
inline signature_t updateSignature(signature_t sig, stride_t str) const {
	sig <<= signatureShift;
	sig ^= str;
	sig &= mask(signatureBits);
	return sig;
}
```

## 2.2. Path Confidence-based Prefetching

### a. 根据新的signature获取PT entry

```cpp
PatternEntry *current_pattern_entry =
    patternTable.findEntry(current_signature, false);
```

### b. 根据prefetch confidence issue prefetch

```cpp
while (current_confidence > lookaheadConfidenceThreshold) {
	...
	// 找到了pattern entry
	if (current_pattern_entry != nullptr) {
		unsigned long max_counter = 0;
		// 遍历PT entry的每个stride, 如果confidence大于threshold, 就issue
		for (auto const &entry : current_pattern_entry->strideEntries) {
			...
			double prefetch_confidence =
				calculatePrefetchConfidence(*current_pattern_entry, entry);

			if (prefetch_confidence >= prefetchConfidenceThreshold) {
				assert(entry.stride != 0);
				//prefetch candidate
				addPrefetch(ppn, current_stride, entry.stride,
							current_confidence, current_signature,
							is_secure, addresses);
			}
		}
	}
	...
}
```

prefetch confidence相关函数

```cpp
double
SignaturePathV2::calculatePrefetchConfidence(PatternEntry const &sig,
        PatternStrideEntry const &entry) const {
    if (sig.counter == 0) return 0.0;
    return ((double) entry.counter) / sig.counter;
}
```

### c. 根据lookahead 加深预取

```cpp
// Look for prefetch candidates while the current path confidence is
// high enough
while (current_confidence > lookaheadConfidenceThreshold) {
	...
	PatternStrideEntry const *lookahead = nullptr;
	if (current_pattern_entry != nullptr) {
		unsigned long max_counter = 0;
		for (auto const &entry : current_pattern_entry->strideEntries) {
			//select the entry with the maximum counter value as lookahead
			if (max_counter < entry.counter) {
				max_counter = entry.counter;
				lookahead = &entry;
			}
			double prefetch_confidence ...
		}
	}

	if (lookahead != nullptr) {
		current_confidence *= calculateLookaheadConfidence(
				*current_pattern_entry, *lookahead);
		// 计算speculative signature
		current_signature =
			updateSignature(current_signature, lookahead->stride);
		current_stride += lookahead->stride;
	} else {
		current_confidence = 0.0;
	}
}
```

计算lookahead confidence的函数: 根据accuracy去throttle

```cpp
double
SignaturePathV2::calculateLookaheadConfidence(
        PatternEntry const &sig, PatternStrideEntry const &lookahead) const {
    if (sig.counter == 0) return 0.0;
    return (((double) usefulPrefetches) / issuedPrefetches) *
            (((double) lookahead.counter) / sig.counter);
}
```

### d. 处理cross page boundary的问题

addPrefetch函数中需要处理cross page boundary的情况

将GHR entry需要的内容记录下来

```cpp
void
SignaturePathV2::handlePageCrossingLookahead(signature_t signature,
            stride_t last_offset, stride_t delta, double path_confidence) {
    // Always use the replacement policy to assign new entries, as all
    // of them are unique, there are never "hits" in the GHR
    GlobalHistoryEntry *gh_entry = globalHistoryRegister.findVictim(0);
    assert(gh_entry != nullptr);
    // Any address value works, as it is never used
    globalHistoryRegister.insertEntry(0, false, gh_entry);

    gh_entry->signature = signature;
    gh_entry->lastBlock = last_offset;
    gh_entry->delta = delta;
    gh_entry->confidence = path_confidence;
}
```

# 3. SPP throttling slides

[SPP throttling pptx](../attachments/SPP_throttling.pptx)
