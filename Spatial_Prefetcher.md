# Spatial Prefetcher

# SPP (Signature Path Prefethcer)
进行了一系列内存访问之后，接下来我可能访问哪些数据。
这个一系列内存访问被压缩成一个signature，一个signature会对应几条可能路径（接下来有多大可能性访问A，有多大可能性访问B...)
<img src="img\Pasted image 20230302160709.png">

## Components

* Signature Table: to capture memory access patterns within 4KB physical pages and to compress the previous deltas in that page into a 12-bit history signature
* Pattern Table
* Prefetch Filter
* Global History Register: cross-page boundary bootstrapped learning
---
## Process

### 1. Learning Memory Access Patterns
### ST存的是什么
ST里面存了256个最近访问的page, 每个page对应一个最后访问的相对于这个page起始地址的offset(存这个是为了计算delta), 以及一个delta signature。这个delta signature是为了访问和更新PT的
### PT存的是什么
PT里面存的是signature，每个signature里面放了几个delta，并给出一个counter来统计可能性，每个signature也会存一个counter用来替换
### 流程
1. 访问ST。L2 cache access来了之后，physical addr传入ST, page number为3, page offset为3。根据他的page number找到对应的entry。可以读到signature是0x1。
2. 访问以及更新PT。计算delta = 3 - 1 = 2, 可以推断一系列访问使得signature为1后，会有一个delta为2的访问。由这个signature和delta访问PT，给他们增加置信度，也就是counter的值。如果发生了miss可以做replacement
3. 更新ST。根据新的delta (+2)来更新signature。

$$New Sig=(OldSig\ll 3)\text{XOR}(Delta)$$

<img src="img\Pasted image 20230302161042.png">

<img src="img\Pasted image 20230302162819.png">

### 2. Path Confidence-based Prefetching
* 更新ST后，得到新的signature A, 访问PT。path confidence为$C_{delta}/C_{sig}$，这里P0为0.8
* 判断path confidence是否大于某个threshold。如果大于
	* PT将当前访问的cache line的base addr加上delta去issue prefetch requests
	* lookahead。PT用当前的signature以及delta计算得到一个speculative lookahead signature，这里是0x52。然后用这个signature去访问PT，选择confidence最高的delta去prefetch。
	* 循环循环。如果confidence低于threshold了就停止。注意confidence是要和前面的相乘的

> [!NOTE]
> SPP使用了一个global accuracy scaling factor $\alpha$来调节lookahead的aggressiveness。
> Path confidence的计算公式为
> $$P_{d}=\alpha \cdot C_{d}\cdot P_{d-1}\;(\alpha<1)$$
> 如果prefetch accuracy比较高，那么$\alpha$会调节值使得Path confidence降低的比较慢。反之同理
>


<img src="img\Pasted image 20230302164436.png">


<img src="img\Pasted image 20230302164451.png">


在两种情况下，SPP会停止prefetch
1. Low path confidence $P_d$
2. Too few L2 read queue resources

### 3. Page Boundary Learning
增加一个记录global history的结构：Global History Register 
当SPP做了一个超出page end的prediction, GHR会被更新；当访问一个新的page, GHR会被查询

<img src="img\Pasted image 20230324163920.png">

* A prefetch request goes beyond the current Page A --> Store in GHR
> GHR可以存储8个entry，每个entry存储了当前的signature, path confidence, last offset, 以及delta。
* New page access
> 如果访问一个new page(ST miss), SPP会先在GHR搜索能跟当前访问match的GHR entry。
比如Figure7 (b)显示的，Last Offset + Delta - 64和当前对page B offset 1的访问对应。因此可以预测，Page B会产生一个对应0x52 signature的pattern
然后用0x52和delta(+3)产生一个Page B的新signature 0x293


### 4. Prefetch Filter
Prefetch Filter是一个direct-mapped filter, 记录当前prefetched cache lines 
SPP在issue prefetch之前会先检查PF。如果PF中存在某个cache line, 那么说明这个line早就被预取了，SPP会丢弃这个redundant prefetch request。

> 在PF中为每个entry增加useful是为了SPP计算accuracy
> SPP会记录两个global counter: $C_{total}$ 记录prefetch request的总数，$C_{useful}$ 记录 useful prefetch的数量。
> $C_{total}$ 在SPP issue一个没有被filter的prefetch request的时候会+1
> $C_{useful}$ 在L2 demand requests hit PF的时候会+1。在PF中增加useful bit是为了防止同一个cache line加多次。

<img src="img\Pasted image 20230324173328.png">


# SPP + PPF

[[todo!]]

[PPF\_Design.svg](https://raw.githubusercontent.com/elbrandt/CS752_Proj/546a5d0602211fcf8b93492e3cabf61dce6194c0/reports/final/PPF_Design.svg)

[H11-Make Prefetch Great Again / Improve Data Prefetching: Perceptron-based Filter in gem5 on Vimeo](https://vimeo.com/543692181)

# DSPatch 

[[to improve!]]

性能表现
 * 在75个不同的工作负载中，仅使用3.6KB的存储空间，DSPatch相对于一个PC-based stride prefetcher在L1缓存和SPP在L2缓存的激进基线提高了6%的性能（在内存密集型工作负载中提高了9%，最高可达26%）。 
 * 作为一个独立的预取器，DSPatch的性能略高于最先进的SPP(1%左右)，且仅需SPP存储要求的2/3。 
 * SPP和DSPatch的使用结合了最先进的delta-based prefetching和bit-pattern-based prefetching的优点。通过同时优化覆盖率和准确性，DSPatch每增加2%的覆盖率只会增加1%的错误预测。最后，DSPatch+SPP的性能随着内存带宽的增加而扩展得很好，从SPP上升6%到DRAM带宽翻倍时上升10%
## A. background
### i. Address access patterns 
* full addresses
* offset in a spatial region (typically a 4KB page)
* address deltas between consecutive accesses
### ii. Prefetcher Comparison
#### SPP: delta-based prefetcher 

SPP uses a recursive look-ahead mechanism to boost prefetch distance and timeliness.

disadvantages: In pages with sparse and highly irregular access patterns,SPP cannot track all possible deltas, losing out on coverage. Thedeltas it tracks have low confidence values, limiting the recursiveprefetch distance and hence timeliness.

### BOP: delta-based prefetcher
Best Offset Prefetcher是一种硬件预取器，它可以动态地选择最佳的预取偏移量，从而提高缓存的命中率和性能。根据我从网络上搜索到的信息，Best Offset Prefetcher的工作原理如下：

- Best Offset Prefetcher在每次缓存未命中或预取命中时，都会生成一个预取请求，并将其发送到下一级缓存。预取地址是通过在访问地址上加上一个偏移量得到的。
- Best Offset Prefetcher使用一个计数器数组来记录每个偏移量的效果。每个计数器对应一个偏移量，初始值为0。当一个偏移量产生了有效的预取时，对应的计数器加1；当一个偏移量产生了无效或过时的预取时，对应的计数器减1。
- Best Offset Prefetcher使用一个寄存器来保存当前选择的最佳偏移量。每隔一段时间（例如1000个周期），Best Offset Prefetcher会扫描计数器数组，找出最大值，并将其对应的偏移量写入寄存器。这样就可以动态地调整最佳偏移量。
- Best Offset Prefetcher还使用了一些技术来优化性能，例如限制预取距离、过滤重复请求、避免干扰正常访问等。

和SPP相比，SPP使用了local view of access, 但是BOP使用了global view of accesses。这给BOP带来了两个好处
* the global view exposes more patterns in a memory region than a restricted local view of consecutive accesses.
	* This especially helps BOP to predict future accesses in workloads with **irregular access patterns** with only few accesses per page.
* the global view is robust against program access reordering, which can further disrupt the pattern learning based on a restricted local view of accesses.

Disadvantages:
* BOP learns only a limited set of global deltas for all access streams in the program in a statically-defined epoch (de￿nedby the number of accesses). This severely limits BOP's **coverage** and **timeliness**

### SMS
- SMS是一种基于代码相关性的预取器，它可以检测出程序访问在一个物理页上的空间相关性，并生成一个偏移量来预取相邻的块。SMS使用一个表来存储每个代码位置对应的最佳偏移量，并且使用一个简单的算法来检测空间相关性。该算法维护两个寄存器：Last PC和Last Address，分别记录上一次访问的代码位置和地址。当发生缓存未命中或预取命中时，SMS会计算当前PC和Last PC之间的差值（Delta PC），以及当前Address和Last Address之间的差值（Delta Address）。如果Delta PC等于0或1，并且Delta Address不等于0或Cache Block Size，则认为存在空间相关性，并将Delta Address作为偏移量生成预取请求。
- SMS有以下优点：（1）它可以利用程序访问在一个物理页上的空间变化，从而提高缓存命中率。（2）它不需要额外的硬件支持，只需要一个表和两个寄存器。（3）它不需要对程序进行修改或注释，只需要对编译器进行简单的修改。
- SMS有以下缺点：（1）它不能利用程序访问在不同物理页上重复出现的空间位模式。（2）它不能根据DRAM带宽利用率动态调整预取数量和覆盖率。
- DSPatch与SMS不同之处在于：（1）DSPatch不仅考虑了空间相关性，还考虑了时间相关性，即程序访问在不同物理页上重复出现的空间位模式。（2）DSPatch使用两个空间位模式来表示一个物理页上的访问布局，分别偏向于覆盖率和准确率，并且根据DRAM带宽利用率动态选择其中一个。（3）DSPatch可以作为SPP的辅助空间预取器，从而提高SPP的预取效果和性能

## DSPatch Design
[https://people.inf.ethz.ch/omutlu/pub/DSPatch\_prefetcher\_micro19-talk.pdf](https://people.inf.ethz.ch/omutlu/pub/DSPatch_prefetcher_micro19-talk.pdf)

The key goal of DSPatch is to dynamically adapt prefetch-ing for either higher coverage or higher accuracy depending onthe DRAM bandwidth utilization.

DSPatch为每个physical page学习两个pattern, 并将这两个pattern和PC based signature相联。这两个pattern分别为
* CovP: for higher coverage, 通过OR操作获得
* AccP: for higher accuracy，通过AND操作获得

### Structure
* Page Buffer (PB)
	* 目的是记录当程序访问一个physical page的时候，记录spatial bit patterns
<img src="img\Pasted image 20230320152726.png">

* Signature Prediction Table (SPT)
	* 目的是存储CovP和AccP, 这两个pattern是从之前的访问中通过一定计算获得的
<img src="img\Pasted image 20230320152736.png">



<img src="img\Pasted image 20230320113534.png">

### Overall Steps
* step 1: Each PB entry tracks accesses in a 4KB physical page and accumulates L1 misses in the page's stored bit-pattern
<img src="img\Pasted image 20230320152840.png">

* step 2: The first access to each 2KB segment in the 4KB physical page is eligible to trigger prefetches. 
<img src="img\Pasted image 20230320152902.png">

* step 3: The PC of this trigger access is stored in the PB entry and used to index into the SPT, which retrieves the two CovP and AccP bit-patterns and the measure of their goodness.
<img src="img\Pasted image 20230320152923.png">

* step 4: Selection logic uses the memory bandwidth utilization measure to select a bit-pattern to generate prefetcher candidates. The selected bit-pattern is anchored(i.e. rotated) to align to the trigger access offset before issuing prefetches.
<img src="img\Pasted image 20230320152941.png">

* step 5: On eviction from the PB, for each trigger (per 2KB segment), the stored bit-pattern is first anchored to trigger offset. Then, SPT is looked up using the stored trigger PC and the stored bit-patterns and the counters are updated.
<img src="img\Pasted image 20230320153039.png">




### Tracking Bandwidth Utilization 
文章提出了一个简单的方法来跟踪DRAM带宽利用率。
在每个内存控制器中维护一个计数器，记录每个内存周期内发出的请求数量。
当计数器超过一个阈值时，认为DRAM带宽已经饱和，此时应该使用偏向于准确率的bit pattern；
当计数器低于另一个阈值时，认为DRAM带宽有空闲，此时应该使用偏向于覆盖率的bit pattern。

### Anchored Spatial Bit-patterns 
DSPatch uses a program access representation that is robust against reordering of accesses in the processor and the memory hierarchy

Use spatial bit-patterns anchored to the trigger (i.e.,first) access to a memory region to capture all local and global deltas from the trigger. (所有都记录相对于在该page上第一个access也就是trigger access的偏移)
<img src="img\Pasted image 20230324105249.png">


### The Choice of Signature and Signature-Pattern Mapping


### Quantifying Accuracy and Coverage 
PopCount of the predicted bit-pattern gives the prefetch count ($C_{pred}$)
PopCount of the access bit-pattern generated by the program gives the total number of accesses ($C_{real}$)
PopCount of the bitwise AND operation between the program bit-pattern and the predicted bit-pattern gives the accurate prefetch count ($C_{acc}$).
**Prediction accuracy**： $C_{acc}/C_{pred}$
**Prediction coverage**: $C_{acc}/C_{real}$
<img src="img\Pasted image 20230320141114.png">


<img src="img\Pasted image 20230320153245.png">


### Modulated Dual Bit-patterns: Coverage-biased and Accuracy-biased
**Coverage-biased Bit-pattern (CovP)**: 

**Accuracy-biased Bit-pattern (AccP)**

<img src="img\Pasted image 20230320143442.png">


<img src="img\Pasted image 20230320144503.png">
