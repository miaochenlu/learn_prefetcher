# Spatial Prefetcher

# 0. Spatial Pattern

- Definition: similar access patterns occur in different regions of memory (e.g., if a program visits locations {A, B, C, D} of Page X, it is probable that it visits locations {A, B, C, D} of other pages as well).
- Source: Spatial correlation transpires because applications use various objects with a regular and fixed layout, and accesses reappear while traversing data structures

<div align="center"><img src="attachments/Pasted%20image%2020230525211309.png" width="400"></div>

# 1. Offset-based Prefetcher

Offset prefetching是next-line prefetching的一般化情况。当core对line X发出Request的时候，offset prefetcher会prefetch line X+D, 其中D就是prefetch offset

![](attachments/Pasted%20image%2020230606143246.png)

## 1.1 BOP (Best-Offset Prefetcher)

---

* [Best-offset hardware prefetching](https://ieeexplore.ieee.org/document/7446087)
* [slides](https://inria.hal.science/hal-01254863/file/BOP_HPCA_2016.pdf)

---

* 不同的benchmark的特性不同，同一benchmark不同程序段的特性也不同，因此有不同的最佳offset，所以应该去动态调节offset的大小。
* BOP对offset的调节中将timeliness作为一个非常重要的因素，它在coverage和timeliness中选择了timeliness

![](attachments/Pasted%20image%2020230606143652.png)

### a. Components
![](attachments/Pasted%20image%2020230606145707.png)
* Recent Requests Table (RR Table): 保存recent requests
BOP想要找到最佳的offset。BOP认为如果line X被访问的时候，line X-d在不久以前被访问，那么offset d被认为是好的offset。最理想的情况是，X-d和X的访问之前间隔的时候和完成一个Prefetch request的latency差不多，那么line X的访问就是timely的。为了实现这个目标，BOP用RR Table将X-d记录下来作为一个recent request。
* Best-offset Learning Unit: 包含offset list & score table，记录offset的值和目前的得分。并不是每一个offset都会被放入offset table, 需要挑选来压缩storage

### b. Process

triggering event: cache miss & prefetch hit。
主要过程分为两步：一个是offset learning，学习best offset; 另一个是prefetching, 实际去issue prefetch request

#### offset learning phase
每个learning phase, BOP会学习得到当前最适合的prefetch offset并更新。
learning phase包括几个阶段
* 首先，所有offset的score置零
* L2 miss或者prefetch hit发生时，对offset list中的offset做遍历。假设$$d_i$$是某个offset list中的某个offset, 如果$$X-d_i$$在RR Table中，说明$$d_i$$是一个能够产生timely prefetch的offset, 给他的score+1。
* 当两个条件发生时，learning phase结束, 更新prefetch offset
	* 某个offset的score达到了SCOREMAX
	* 学习的rounds达到了ROUNDMAX
![](attachments/Pasted%20image%2020230606154912.png)
#### prefetching
假设当前learning得到的best offset为D，如果D的score低于BAD_SCORE，则关闭prefetch(仍然train)。
如果prefetch开启，当line X的triggering event发生时，计算prefetch line为X+D, 如果X和X+D在同一个page的话，就issue prefetch request
![](attachments/Pasted%20image%2020230606154613.png)

当prefetch request完成，将prefetch request的base addr存到RR table
![](attachments/Pasted%20image%2020230606155452.png)

### c. Overhead
![](attachments/Pasted%20image%2020230606160324.png)

### D. Other
#### Offset的选择
论文中选择了1到256中质因数分解后所有质因数都不大于5的数
> 1 2 3 4 5 6 8 9 10 12 15 16 18 20 24 25 27 30 32 36 40 45 48 50 54 60 64 72 75 80 81 90 96 100 108 120 125 128 135 144 150 160 162 180 192 200 216 225 240 243 250 256

## 1.2 MLOP (Multi-Lookahead Offset Prefetcher)

---

* [Multi-Lookahead Offset Prefetching](https://mshakerinava.github.io/papers/mlop-dpc3.pdf)
* [slides](https://dpc3.compas.cs.stonybrook.edu/slides/mlop_dpc3.pptx)

---

之前的Offset Prefetcher, 比如Sanbox Prefetcher, BOP, 要么牺牲了timeliness, 要么牺牲了coverage。MLOP既要又要。
BOP在prefetch的时候只考虑一个全局最佳的offset，导致错过了很多预取机会，MLOP提出可以有多个lookahead level，每个lookahead level有自己的最佳全局offset。

解释一下lookahead level, 在lookahead level X的选中了某个Offset, 代表offset prefetcher至少在X次access前prefetch当前访问才能prefetch hit

### a. Components
#### a.i Access Map Table
AMT中存放的是一个一个region的memory访问情况
每个region对应一个entry, entry中存储一个bit vector, 表示每一个位置是否被访问过了。
此外，每个region还会有一个history queue。以queue的形式存最近k个访问的offset，支持lookahead level。

#### a.ii Offset Score Table
因为MLOP有多个lookahead level, 每个level有自己的最佳offset。因此，MLOP开了level个score table, 记录当前level每个offset的得分

![](attachments/Pasted%20image%2020230720134736.png)
### b. Process
* Initial state: history queue没有数据，access map没有被访问
![](attachments/Pasted%20image%2020230720140547.png)
* Access line (offset 0)：history queue push进0，access map bit 0置为ACCESS
![](attachments/Pasted%20image%2020230720140600.png)
* Access line (offset 1)：history queue push进1，access map bit 1置为ACCESS
![](attachments/Pasted%20image%2020230720140613.png)
* Access line (offset 3): history queue push进3，access map bit 3置为ACCESS
![](attachments/Pasted%20image%2020230720140626.png)
* Access line (offset 4): history queue push进4, 0被pop，access map bit 4置为ACCESS
![](attachments/Pasted%20image%2020230720140652.png)
* Access line (offset 6): history queue push进6, 1被pop，access map bit 6置为ACCESS

![](attachments/Pasted%20image%2020230720140706.png)
* Score update (以accsss line 6为例): 在lookahead level 1, 也就是1次access前prefetch当前访问才能prefetch hit，于是将当前access map中的已经访问的位置与当前access的位置去做差值，对这些offset加分。在lookahead level 2, 也就是2次access前prefetch当前访问才能prefetch hit, 所以要把前1次访问unmark了，也就是把bit 6 unmark了，然后再来update score。之后的level同理
![](attachments/Pasted%20image%2020230720140728.png)

![](attachments/Pasted%20image%2020230720140759.png)
![](attachments/Pasted%20image%2020230720140824.png)
![](attachments/Pasted%20image%2020230720140841.png)
* 训练达到round之后，每个level会有自己的最佳offset, 在prefetch的时候issue出去
### c. Settings & Overhead
* Sits in L1 and prefetches into L1 and L2.
* Evaluates all possible offsets ([-63, +63]).
* Uses lookahead levels 1 to 16.
* Updates scores 500 times in each round.
* Selected offsets with score above 200 prefetch into L1. And selected offsets with score above 150 (but not above 200) prefetch into L2.

# 2. Delta-based Prefetcher

## 2.1 SPP (Signature Path Prefetcher)

---

- [Path confidence based lookahead prefetching](https://ieeexplore.ieee.org/document/7783763/)
- [gem5/signature\_path\_v2.hh at stable · gem5/gem5 · GitHub](https://github.com/gem5/gem5/blob/stable/src/mem/cache/prefetch/signature_path_v2.hh)
- [SPP code reading](simulator_codes/SPP_Gem5_Codes.md)
- [SPP_throttling.pptx](attachments/SPP_throttling.pptx)

---

prefetcher主要是为了在进行了一系列内存访问之后，预测接下来我可能访问哪些数据。
SPP将这个一系列内存访问被压缩成一个signature，一个signature会对应几条可能路径（接下来有多大可能性访问A，有多大可能性访问B...），SPP会prefetch这些路径上接下来可能被访问的数据

SPP通过compressed history signature能够prefetch复杂的memory access pattern; 使用了lookahead机制提升了prefetch的及时性; 支持跨page boundary的prefetch; 通过path confidence来调控prefetch的激进程度

<div align="center"><img src="attachments/Pasted%20image%2020230424102912.png" width="400"></div>

### a. Main Components

- Signature Table: to capture memory access patterns within 4KB physical pages and to compress the previous deltas in that page into a 12-bit history signature
- Pattern Table
- Prefetch Filter
- Global History Register: cross-page boundary bootstrapped learning

<div align="center"><img src="attachments/Pasted%20image%2020230302160709.png" width="300"></div>

### b. Process

#### b.i. Learning Memory Access Patterns

##### <font color="#2DC26B">ST存的是什么</font>

ST里面存了最近访问的page的一些信息

ST会记录这些page最后访问的offset(存这个是为了计算delta), 以及一个delta signature(根据page访问pattern压缩成的signature)。通过这个delta signature访问PT

##### <font color="#2DC26B">PT存的是什么</font>

PT里面存的是每个signature之后可能访问的stride的信息。

在PT中每个signature会有一个counter计算置信程度，每个signature会对应几个delta，每个delta也会对应一个counter来统计置信程度

##### <font color="#2DC26B">流程</font>

以下图为例

1. 访问ST。L2 cache access来了之后，physical addr传入ST, page number为3, page offset为3。根据他的page number找到对应的entry。可以读到signature是0x1。
2. 访问以及更新PT。计算delta = 3 - 1 = 2, 可以推断一系列访问使得signature为1后，接下来可能会有一个delta为2的访问。由这个signature和delta访问PT，给他们增加置信度，也就是counter的值，于是PT中signature为0x1的entry的$$C_{sig}$$ counter增加并且Delta为2的counter增加。如果发生了miss可以做replacement
3. 更新ST。根据新的delta (+2)来更新signature。
$$New Sig=(OldSig\ll 3)\text{XOR}(Delta)$$

<div align="center"><img src="attachments/Pasted%20image%2020230302161042.png" width="300"></div>

<div align="center"><img src="attachments/Pasted%20image%2020230302162819.png" width="300"></div>

#### b.ii. Path Confidence-based Prefetching

- 更新ST后，得到新的signature A, 访问PT。path confidence为$$C_{delta}/C_{sig}$$，这里P0为$$8/10=0.8$$

- 判断path confidence是否大于设定的threshold。如果大于
  - PT将当前访问的cache line的base addr加上delta去issue prefetch requests
  - lookahead。PT用当前的signature以及delta计算得到一个speculative lookahead signature，这里是0x52。然后用这个signature去访问PT，选择confidence最高的delta去prefetch。
  - 循环循环。如果confidence低于threshold了就停止。注意confidence是要和前面的相乘的

> [!NOTE]
> 
> SPP使用了一个global accuracy scaling factor $$\alpha$$来调节lookahead的aggressiveness。
> 
> Path confidence的计算公式为
> 
> $$P_{d}=\alpha \cdot C_{d}\cdot P_{d-1} (\alpha<1)$$
> 
> 如果prefetch accuracy比较高，那么$$\alpha$$会调节值使得Path confidence降低的比较慢。反之同理
>

<div align="center"><img src="attachments/Pasted%20image%2020230302164436.png" width="300"></div>

<div align="center"><img src="attachments/Pasted%20image%2020230302164451.png" width="300"></div>

在两种情况下，SPP会停止prefetch

1. Low path confidence $$P_d$$
2. Too few L2 read queue resources

#### b.iii. Page Boundary Learning

增加一个记录global history的结构：Global History Register

当SPP做了一个超出page end的prediction, GHR会被更新；当访问一个新的page, GHR会被查询

<div align="center"><img src="attachments/Pasted%20image%2020230324163920.png" width="300"></div>

- A prefetch request goes beyond the current Page A --> Store in GHR

> GHR可以存储8个entry，每个entry存储了当前的signature, path confidence, last offset, 以及delta。

- New page access

> 如果访问一个new page(ST miss), SPP会先在GHR搜索能跟当前访问match的GHR entry。

比如Figure7 (b)显示的，Last Offset + Delta - 64和当前对page B offset 1的访问对应。因此可以预测，Page B会产生一个对应0x52 signature的pattern

然后用0x52和delta(+3)产生一个Page B的新signature 0x293

### b.iv. Prefetch Filter

Prefetch Filter是一个direct-mapped filter, 记录当前prefetched cache lines
SPP在issue prefetch之前会先检查PF。如果PF中存在某个cache line, 那么说明这个line早就被预取了，SPP会丢弃这个redundant prefetch request。

> 在PF中为每个entry增加useful是为了SPP计算accuracy
>
> SPP会记录两个global counter: $$C_{total}$$ 记录prefetch request的总数，$$C_{useful}$$ 记录 useful prefetch的数量。
>
> $$C_{total}$$ 在SPP issue一个没有被filter的prefetch request的时候会+1
>
>$$C_{useful}$$ 在L2 demand requests hit PF的时候会+1。在PF中增加useful bit是为了防止同一个cache line加多次。

<div align="center"><img src="attachments/Pasted%20image%2020230324173328.png" width="300"></div>

### c. Overhead

<div align="center"><img src="attachments/Pasted%20image%2020230526105701.png" width="300"></div>

## 2.2 SPP + PPF

---

- [ISCA'19 Perceptron-Based Prefetch Filtering](https://people.engr.tamu.edu/djimenez/pdfs/ppf_isca2019.pdf)
- [PPF src codes](https://github.com/agusnt/Berti-Artifact/blob/main/ChampSim/Other_PF/prefetcher/ppf.l2c_pref)
- [SPP + PPF ChampSim Codes](simulator_codes/SPPPPF_ChampSim_Codes.md)

---

目的：SPP目前的confidence-based throttling机制很复杂。accuracy和coverage是两个此消彼长的因素，很难调节。PPF可以让SPP激进的发送请求，并且将其的一些无用的request过滤掉，达到兼顾coverage和accuracy的效果

### a. Structure

#### a.i. Perceptron

<div align="center"><img src="attachments/Pasted%20image%2020230404164941.png" width="200"></div>

对于一个有N个feature的PPF来说，他包含

- N个table（存储权重）
  - table里面的每个entry代表一个权重。权哥entry是一个5位的饱和计数器

#### a.ii. Prefetch Table & Reject Table

这两个Table中Prefetch Table记录了prefetch request的信息，Reject Table记录了不应该被prefetch的信息，这两个table被用来train perceptron

结构上，他们都是直接映射的结构，有1024个entry, 用addr的10位地址索引，6位做tag比较，entry中存储了各种feature相关的信息

<div align="center"><img src="attachments/Pasted%20image%2020230404164953.png" width="200"></div>

### b. Process

#### b.i. Interfencing

对于suggested prefetch request, 先通过perceptron中各种feature的weight计算出sum。将sum和两个threshold作比较： $$\tau_{hi}$$ and $$\tau_{lo}$$

> $$sum>\tau_{hi}$$ -> prefetch into L2
> 
> $$\tau_{lo}<=sum<=\tau_{hi}$$ -> prefetch into LLC

<div align="center"><img src="attachments/Pasted%20image%2020230404112020.png" width="400"></div>

#### b.ii. Recording

过程中将认为应该issue到L2的prefetch的加入prefetch table, 将其他的request加入reject table

#### b.iii. Training

主要在两个时间点进行PPF training, 一个是demand request来的时候，另一个是cache eviction的时候。

Demand request来的时候，访问prefetch table更新权重，访问reject table更新权重

同样cache eviction的时候，也做权重的更新

### c. Overhead

<div align="center"><img src="attachments/Pasted%20image%2020230529212519.png" width="300"></div>

<div align="center"><img src="attachments/Pasted%20image%2020230529212528.png" width="300"></div>

# 3. Bit-pattern-based Prefetcher

## 3.1. SMS (Spatial Memory Streaming)

---

- [Spatial Memory Streaming](https://web.eecs.umich.edu/~twenisch/papers/isca06.pdf)

---

SMS的主要思想是track一个memory region上的访问模式，形成一个bit pattern。

SMS认为如果指令PC和trigger offset一样的话，访问其他的region也应该是类似的bit-pattern (Trigger offset是在这个region上第一次访问时的offset)。

也就是说SMS给这个bit pattern一个signature, 这个signature是由PC+trigger offset构成，signature相同的访问对应相同的bit-pattern。

<div align="center"><img src="attachments/Pasted%20image%2020230424103000.png" width="400"></div>

### a. Main Components & Process

SMS最主要的结构有Active Generation Table，包含Accumulation Table和Filter Table。region第一次被访问时，会在FT中allocate一个entry，而不会直接在AT中记录pattern, 这样可以过滤掉一些只被访问一次的region。AT记录memory region的bit-pattern。当region的pattern训练成熟，也就是被evict的时候，会放入Pattern History Table。

<div align="center"><img src="attachments/Pasted%20image%2020230424103335.png" width="400"></div>

Pattern History Table使用PC+offset索引，然后看bit-pattern中哪些位置1了，就对该位置对应的addr进行prefetch

<div align="center"><img src="attachments/Pasted%20image%2020230424103340.png" width="400"></div>

下面是一个更细节的organization图

<div align="center"><img src="attachments/Pasted%20image%2020230526144946.png" width="400"></div>

### c. Overhead & Performance

SMS的性能取决于PHT存储的pattern的数量
下图是SMS的PHT的size变化对性能的影响。其中PHT的size从16K entry(16-way assoc) total 88KB变化到256 entry total 3.5KB

<div align="center"><img src="attachments/Pasted%20image%2020230526145446.png" width="400"></div>

## 3.2. PMP

---

- [MICRO'22 Merging Similar Patterns for Hardware Prefetching](https://ieeexplore.ieee.org/document/9923831)

---

如在SMS中所说，SMS的性能和storage overhead存在一个trade-off，如果storage比较小的话，SMS的性能会急速下降。PMP主要是想在减少SMS的storage的同时获得high performance。

PMP主要做了以下工作

- 分析Bit-pattern的signature应该选取哪些指标比较好
- 做了pattern merging来减少存储开销

## a. Feature analysis (How to Form the Signature)

SMS用了PC+trigger offset作为匹配bit pattern的signature
其实还可以选择其他特征，比如用单独pc去匹配，单独用trigger offset, 或者用memory address等等

PMP论文中提出了评估这些feature好坏的两个指标，一个是Pattern collision rate, 一个是pattern duplicate rate

- **Pattern collision rate**是说同一个feature，可能会对应很多不同的bit pattern，导致pattern table的bit pattern老是被替换
- **Pattern duplicate rate**是在一个feature在不同取值情况下，bit pattern是否出现重复，PDR这个越大代表存储的冗余越多

<div align="center"><img src="attachments/Pasted%20image%2020230424103529.png" width="300"></div>

论文中检测了很多feature组合的PCR和PDR，如下表所示

PMP选择了冗余比较少的trigger offset作为一个匹配的feature。同时为了减少trigger offset高PCR的影响，提出了一种merge的方法。比如一个trigger offset对应了几个不同的bit pattern，PMP不是不断的去替换成最新的bit pattern而是将他们merge起来成为一个新的Pattern

<div align="center"><img src="attachments/Pasted%20image%2020230424103535.png" width="400"></div>

### b. PMP整体框架

保留了SMS的Active Generation Table，包含Accumulation Table和Filter Table。将Pattern History Table替换成了Offset Pattern Table和PC Pattern Table, 分别以Offset和PC作为pattern的signature。Pattern Table中的每一位从一个Bit变成了一个counter

<div align="center"><img src="attachments/Pasted%20image%2020230424103555.png" width="800"></div>

图左边展示了offset pattern的merge过程。如果某个offset新来了一个bit pattern，PMP先根据trigger offset进行一次anchor, 然后和之前的bit pattern相加。

在prefetch的时候，如果counter大于某一置信程度，就会在L1预取，置信程度一般的话发给让L2预取。

在trigger offset这个特征的基础上，PMP还加入了一个辅助特征PC, 他会对预取level的决策产生一些影响

### c. overhead

<div align="center"><img src="attachments/Pasted%20image%2020230526144412.png" width="300"></div>

## 3.3. DSPatch

---

- [MICRO'19 DSPatch: Dual Spatial Pattern Prefetcher](https://people.inf.ethz.ch/omutlu/pub/DSPatch_prefetcher_micro19.pdf)
- [slides](https://people.inf.ethz.ch/omutlu/pub/DSPatch_prefetcher_micro19-talk.pdf)
- [DSPatch ChampSim Codes](simulator_codes/DSPatch_ChampSim_Codes.md)

---

DSPatch主要思想是根据DRAM bandwidth的使用情况来调节prefetch是朝着higher coverage还是higher accuracy的方向努力。如果DRAM bandwidth比较低，则倾向于higher coverage; 如果DRAM bandwidth比较高，则倾向于higher accuracy。

DSPatch为每个physical page学习两个pattern, 并将这两个pattern和PC based signature相联。这两个pattern分别为

- CovP: for higher coverage, 通过OR操作获得
- AccP: for higher accuracy，通过AND操作获得

### a. Main Components

- Page Buffer (PB)
  - 当程序访问一个physical page的时候，记录spatial bit patterns

<div align="center"><img src="attachments/Pasted%20image%2020230320152726.png" width="400"></div>

- Signature Prediction Table (SPT)
  - 目的是存储CovP和AccP, 这两个pattern是从之前的访问中通过一定计算获得的(OR & AND)

<div align="center"><img src="attachments/Pasted%20image%2020230320152736.png" width="400"></div>

### b. Process

<div align="center"><img src="attachments/Pasted%20image%2020230320113534.png" width="400"></div>

#### step 1

Each PB entry tracks accesses in a 4KB physical page and accumulates L1 misses in the page's stored bit-pattern

<div align="center"><img src="attachments/Pasted%20image%2020230320152840.png" width="400"></div>

#### step 2

The first access to each 2KB segment in the 4KB physical page is eligible to trigger prefetches.

<div align="center"><img src="attachments/Pasted%20image%2020230320152902.png" width="400"></div>

#### step 3

The PC of this trigger access is stored in the PB entry and used to index into the SPT, which retrieves the two CovP and AccP bit-patterns and the measure of their goodness.

<div align="center"><img src="attachments/Pasted%20image%2020230320152923.png" width="400"></div>

#### step 4

Selection logic uses the memory bandwidth utilization measure to select a bit-pattern to generate prefetcher candidates. The selected bit-pattern is anchored(i.e. rotated) to align to the trigger access offset before issuing prefetches.

<div align="center"><img src="attachments/Pasted%20image%2020230320152941.png" width="400"></div>

#### step 5

On eviction from the PB, for each trigger (per 2KB segment), the stored bit-pattern is first anchored to trigger offset. Then, SPT is looked up using the stored trigger PC and the stored bit-patterns and the counters are updated.

<div align="center"><img src="attachments/Pasted%20image%2020230320153039.png" width="400"></div>

- Anchored Spatial Bit-patterns

以trigger access(first access to a memory region)为锚点, 移动bit-pattern(trigger access作为第一个bit)。这样转换能够将一个region上相对于page start addr的bit-pattern转换成一个delta bit pattern，有助于研究delta关系。MICRO'22的PMP也使用了这种方法。

<div align="center"><img src="attachments/Pasted%20image%2020230324105249.png" width="400"></div>

#### b.i.Tracking Bandwidth Utilization

文章提出了一个简单的方法来跟踪DRAM带宽利用率。

在每个内存控制器中维护一个计数器，记录每个内存周期内发出的请求数量。

当计数器超过一个阈值时，认为DRAM带宽已经饱和，此时应该使用偏向于accuracy的bit pattern。

当计数器低于另一个阈值时，认为DRAM带宽有空闲，此时应该使用偏向于coverage的bit pattern。

#### b.ii Quantifying Accuracy and Coverage

Step 4中，除了dram bandwidth之外，DSPatch还需要考虑SPT中pattern的accuracy和coverage。如果accuracy很差，即使bandwidth有空余也不应该issue prefetch requests.

- PopCount of the predicted bit-pattern gives the prefetch count ($$C_{pred}$$)
- PopCount of the access bit-pattern generated by the program gives the total number of accesses ($$C_{real}$$)
- PopCount of the bitwise AND operation between the program bit-pattern and the predicted bit-pattern gives the accurate prefetch count ($$C_{acc}$$).

根据上述counter计算accuracy和coverage

**Prediction accuracy**： $$C_{acc}/C_{pred}$$

**Prediction coverage**: $$C_{acc}/C_{real}$$

<div align="center"><img src="attachments/Pasted%20image%2020230320141114.png" width="400"></div>

<div align="center"><img src="attachments/Pasted%20image%2020230320153245.png" width="400"></div>

有了上述信息之后，最终按照如下逻辑来判断如何issue prefetch requests

<div align="center"><img src="attachments/Pasted%20image%2020230529164915.png" width="400"></div>

### c. Overhead

<div align="center"><img src="attachments/Pasted%20image%2020230529211750.png" width="400"></div>

# 4. Composite

## 4.1. IPCP Prefetcher

---

- [ISCA'20 Bouquet of Instruction Pointers: Instruction Pointer Classifier-based Spatial Hardware Prefetching](https://www.cse.iitk.ac.in/users/biswap/IPCP_ISCA20.pdf)
- [slides](https://dpc3.compas.cs.stonybrook.edu/slides/bouquet.pdf)
- [IPCP ChampSim Codes](simulator_codes/IPCP_ChampSim_Codes.md)

---

### a. Components: 四类prefetcher

#### a.i. Constant Stride (CS): only control flow

##### <font color="#2DC26B">目标</font>

预取如下呈现的constant stride pattern

<div align="center"><img src="attachments/Pasted%20image%2020230410162234.png" width="400"></div>

##### <font color="#2DC26B">components</font>

下图是constant stride (CS)下IP table的内容

<div align="center"><img src="attachments/Pasted%20image%2020230410164218.png" width="400"></div>

- 由IP来index和tag
- stride是用来记录constant stride的
- confidence来记录当前stride的置信程度。如果遇到了相同的stride, confidence增加，反之减小
- last_vpage存储了上一次访问 的virtual page的2个低位，用来检测page的变化。
- last_line_offset存储了上一次访问的cache line相对于page的offset
  - last_vpage以及last_line_offset二者一起决定了stride的计算结果。比如last page是1，last_line_offset是63，当前page是2，line offset是0，那么当前stride就是(0-63)+64(2-1)=1

##### <font color="#2DC26B">process</font>

- Training phase
  - IP会持续training直到confidence达到某个threshold

- Thained phase
  - IP获得足够的confidence后进入trained状态
  - 发起perfetch请求
    - prefetch addr = (cur cacheline addr) + k * (learned stride), 其中k取1到prefetch degree的值
  - confidence低于threshold进入training状态

#### a.ii. Complex Stride (CPLX): control flow coupled with data flow

##### <font color="#2DC26B">目标</font>

在stride不断变化的情况下取得比较好的预取效果

<div align="center"><img src="attachments/Pasted%20image%2020230410163823.png" width="400"></div>

##### <font color="#2DC26B">components</font>

<div align="center"><img src="attachments/Pasted%20image%2020230410164206.png" width="400"></div>

- IP table
  - 同上的IP table, 加入一个n位的signature, 记录前n个stride hash之后的结果
- CSPT
  - 由signature索引
  - stride是预测的当前的signature下，下一次访问的stride是多少
  - confidence代表这个预测的stride的置信程度

##### <font color="#2DC26B">Process</font>

- Training phase
  - IP对应的signature索引到CSPT
  - 如果CSPT中的stride和当前stride一样，则增加confidence, 反之减少
  - 更新IP table中的signature

- Trained phase
  - 用上述更新过的signature再去访问CSPT，如果对应的stride confidence足够高，则发送预取请求
  - 在prefetch degree下，通过1,2,3步不断发射预取  

##### <font color="#2DC26B">CPLX与SPP的区别</font>

- 关注的点不一样
  - SPP关注某一page的访问
  - CPLX是关注某一IP

- access pattern不一样，以下几种情况CPLX会好一点
  - The memory accesses (for a given IP) are sometimes not in the powers of two (memory layout in data structures across cache lines), causing an nonconstant stride pattern.
    - 例子是consider a cache line of 8 bytes, and if every 12th byte is accessed, the accesses create strides as follows: byte addresses: 0, 12, 36, 48, 72; cache line aligned addresses: 0, 1, 3, 4, 6; strides: 1, 2, 1, 2
  - 对于多重循环的访问
    - An outer loop could make constant stride accesses (can be easily captured by the CS class).
    - However, an inner loop could make different stride accesses (depending on the strides of the outer loop), thus causing bumps in the stride pattern. An IP based CPLX can exploit this pattern.
- global & local
  - SPP关注的是global pattern
  - CPLX关注的是local pattern

#### a.iii. Global stream (GS): control flow predicted data flow

##### <font color="#2DC26B">components</font>

<div align="center"><img src="attachments/Pasted%20image%2020230410203651.png" width="400"></div>

- IP table
  - IP tag与索引
  - stream-valid
  - direction
- Region Stream Table(RST)
  - 记录每个region以及他们的访问，一个region大小为2KB
  - 32-bit bit-vector: 记录该region中32个cache line的访问情况。如果该region中的某个cacheline被第一次访问，对应的bit会置1，同时dense counter加1
  - dense counter:记录不同的cache line的访问次数。当dense-count超过了75%，表示这个region已经训练完成
  - last line offset: 记录region中最后一次访问的offset

##### <font color="#2DC26B">Process</font>

- Training Process
  - 如果一个新的region被访问，则在RST中分配一个新的entry。如果该region中的某个cacheline被第一次访问，对应的bit会置1，同时dense counter加1（不是第一次访问的话不会增加）。
  - 如果dense counter超过75%，则所有访问该region的IP成为GS IP。trained bit被置1
  - RST使用n-bit饱和计数器来决定stream的direction (pos / neg count)。这个计数器被初始化为$$2^n/2$$。通过找到两个连续访问之间的差值计数（last-line-offset起作用），如果是正的插值，则加1，负的插值则减1
  - 如果GS IP到了一个新的region，则通过last-vpage和last-line-offset看他之前访问的region。如果之前访问的region是dense的(trained bit set), 那么将新region的RST entry的tentative bit置1。

- Trained Process
  - demand access来的时候，检查RST entry的trained bit和tentative bit, 如果有一个set了，说明这个IP属于GS IP
  - Set IP table中的stream-valid和direction
  - GS IP根据trained direction prefetch接下来的prefetch degree个cache line

#### a.vi. Next line

如果CS, CPLX, GS不是的话，则启用NL prefetcher。

当MPKI太高的时候，不要开启NL Prefetcher

### b. IPCP整体架构

#### b.i. IPCP at L1

<div align="center"><img src="attachments/Pasted%20image%2020230410210908.png" width="400"></div>

##### <font color="#2DC26B">Components</font>

- IP Table
  - Shared: IP-tag, Valid, last-vpage, last-line-offset
  - for CS:
    - Stride
    - Confidence
  - for CPLX:
    - Signature
  - for GS:
    - Stream valid
    - direction

- RST
- CSPT

关于IP table的替换问题：

>When an IP is encountered for the first time, it is recorded in the IP table and the valid bit is set.
>  
>When another IP maps to the same entry, the valid bit is reset, but the previous entry remains active.
>
>If the valid bit is reset when a new IP is seen then the table entry is allocated to the new IP and the valid bit is set again, ensuring that at least one of the two competing IPs is tracked in the IP table.

##### Process

1. IP table hit, 则IPCP同时检查CS/CPLX/GS。同一时间，IP可能不属于任意一个class, 或者属于多个class。同时，检查RST并且训练
2. IPCP看IP属于CS还是GS。如果都属于，那么IPCP会优先选择GS
3. IPCP不属于CS和GS，看是否属于CPLX。
4. 如果CPLX的confidence不高，则根据MPKI选择NL Prefetcher

整体上，优先级为

$$GS>CS>CPLX>NL$$

##### 一些设计问题

- Prefetch Degree
  - GS: 6
  - CS & CPLX: 3

- Filter
  - Use a small recent-request filter (RR filter, 32 entry) to track recently seen tags.

#### b.ii. IPCP at L2

<div align="center"><img src="attachments/Pasted%20image%2020230411095539.png" width="400"></div>

IPCP at L2没有CPLX, 因为测出来没啥用

##### <font color="#2DC26B">components</font>

- IP table
  - IP tag, IP valid bit
  - 2-bit class type
  - 7-bit stride/direction

##### <font color="#2DC26B">Process</font>

- Metadata Decoding
use the L1 prefetch requests to communicate the IP classification information to the L2 prefetcher by transmitting lightweight metadata along with the prefetch requests.

<div align="center"><img src="attachments/Pasted%20image%2020230411100255.png" width="400"></div>

- Prefetch
  - L2 demand access, 访问L2 IP table

##### <font color="#2DC26B">一些设计问题</font>

- Prefetch Degree
  - GS: 4
  - CS: 4

### c. overhead

<div align="center"><img src="attachments/Pasted%20image%2020230529211647.png" width="400"></div>
