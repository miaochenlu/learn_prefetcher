# ISB Gem5 Code Reading

# 1. Components

## 1.1. Training Unit

```cpp
/**
 * Training Unit Entry datatype, it holds the last accessed address and
 * its secure flag
 */
struct TrainingUnitEntry : public TaggedEntry
{
    Addr lastAddress;
    bool lastAddressSecure;
};
/** Map of PCs to Training unit entries */
AssociativeSet<TrainingUnitEntry> trainingUnit;
```

## 1.2. Address Mapping Caches

<div align="center"><img src="attachments/Pasted%20image%2020230130234740.png" width="300"></div>

Entry的设置

```cpp
/** Address Mapping entry, holds an address and a confidence counter */
struct AddressMapping
{
    Addr address;
    SatCounter8 counter;
    AddressMapping(unsigned bits) : address(0), counter(bits)
    {}
};

/**
 * Maps a set of contiguous addresses to another set of (not necessarily
 * contiguos) addresses, with their corresponding confidence counters
 */
struct AddressMappingEntry : public TaggedEntry
{
	/*********************一个entry中有num_mappings个mapping*************/
    std::vector<AddressMapping> mappings;
    AddressMappingEntry(size_t num_mappings, unsigned counter_bits)
        : TaggedEntry(), mappings(num_mappings, counter_bits)
    {
    }

    void
    invalidate() override
    {
        TaggedEntry::invalidate();
        for (auto &entry : mappings) {
            entry.address = 0;
            entry.counter.reset();
        }
    }
};
```

```cpp
/** Physical-to-Structured mappings table */
AssociativeSet<AddressMappingEntry> psAddressMappingCache;
/** Structured-to-Physical mappings table */
AssociativeSet<AddressMappingEntry> spAddressMappingCache;
```

# 2. Process

## 2.1. Training

```cpp
    // This prefetcher requires a PC
    if (!pfi.hasPC()) {
        return;
    }
    bool is_secure = pfi.isSecure();
    Addr pc = pfi.getPC();
    Addr addr = blockIndex(pfi.getAddr());

    // Training, if the entry exists, then we found a correlation between
    // the entry lastAddress (named as correlated_addr_A) and the address of
    // the current access (named as correlated_addr_B)
    TrainingUnitEntry *entry = trainingUnit.findEntry(pc, is_secure);
    bool correlated_addr_found = false;
    Addr correlated_addr_A = 0;
    Addr correlated_addr_B = 0;
    if (entry != nullptr && entry->lastAddressSecure == is_secure) {
        trainingUnit.accessEntry(entry);
        correlated_addr_found = true;
        correlated_addr_A = entry->lastAddress;
        correlated_addr_B = addr;
    } else {
        entry = trainingUnit.findVictim(pc);
        assert(entry != nullptr);

        trainingUnit.insertEntry(pc, is_secure, entry);
    }
    // Update the entry
    entry->lastAddress = addr;
    entry->lastAddressSecure = is_secure;

```

## 2.2. Physical -> Structural Address Mapping

<div align="center"><img src="attachments/Pasted%20image%2020230131001114.png" width="400"></div>

```cpp
IrregularStreamBuffer::AddressMapping&
IrregularStreamBuffer::getPSMapping(Addr paddr, bool is_secure)
{
    Addr amc_address = paddr / prefetchCandidatesPerEntry;
    Addr map_index   = paddr % prefetchCandidatesPerEntry;
    AddressMappingEntry *ps_entry =
        psAddressMappingCache.findEntry(amc_address, is_secure);
    if (ps_entry != nullptr) {
        // A PS-AMC line already exists
        psAddressMappingCache.accessEntry(ps_entry);
    } else {
        ps_entry = psAddressMappingCache.findVictim(amc_address);
        assert(ps_entry != nullptr);

        psAddressMappingCache.insertEntry(amc_address, is_secure, ps_entry);
    }
    return ps_entry->mappings[map_index];
}
```

## 2.3. Generate Prefetch & Get Structural -> Physical Address Mapping

<div align="center"><img src="attachments/Pasted%20image%2020230131001603.png" width="400"></div>

```cpp
    // Use the PS mapping to predict future accesses using the current address
    // - Look for the structured address
    // - if it exists, use it to generate prefetches for the subsequent
    //   addresses in ascending order, as many as indicated by the degree
    //   (given the structured address S, prefetch S+1, S+2, .. up to S+degree)
    Addr amc_address = addr / prefetchCandidatesPerEntry;
    Addr map_index   = addr % prefetchCandidatesPerEntry;
    AddressMappingEntry *ps_am = psAddressMappingCache.findEntry(amc_address,
                                                                 is_secure);
    if (ps_am != nullptr) {
        AddressMapping &mapping = ps_am->mappings[map_index];
        if (mapping.counter > 0) {
            Addr sp_address = mapping.address / prefetchCandidatesPerEntry;
            Addr sp_index   = mapping.address % prefetchCandidatesPerEntry;
            AddressMappingEntry *sp_am =
                spAddressMappingCache.findEntry(sp_address, is_secure);
            if (sp_am == nullptr) {
                // The entry has been evicted, can not generate prefetches
                return;
            }
            for (unsigned d = 1;
                    d <= degree && (sp_index + d) < prefetchCandidatesPerEntry;
                    d += 1)
            {
                AddressMapping &spm = sp_am->mappings[sp_index + d];
                //generate prefetch
                if (spm.counter > 0) {
                    Addr pf_addr = spm.address << lBlkSize;
                    addresses.push_back(AddrPriority(pf_addr, 0));
                }
            }
        }
    }
```
