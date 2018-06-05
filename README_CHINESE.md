<h1 align="center"><img width="300px" src="doc/img/isoseq3.png"/></h1>
<h1 align="center">IsoSeq3</h1>
<p align="center">Scalable De Novo Isoform Discovery</p>

***

## Scope

*IsoSeq3* 是从 PacBio 单分子测序数据 中识别转录组数据的最新软件。
从SMRTLink V6.0开始，*图形化的Iso-Seq应用协议* 将从底层调用 *IsoSeq3*
库中的软件。

*IsoSeq3* 包含 组件化的工作流程和算法，尤其使用了新的聚类算法，可以快速高效得处理更大通量的PacBio数据，并生成和*IsoSeq1* 以及 *IsoSeq2* 相似的高质量的转录组分析结果。


## 概述
 - [SMRTbell 设计](README_CHINESE.md#smrtbell-设计)
 - [流程 概述](README_CHINESE.md#流程)
 - [安装](README_CHINESE.md#安装)
 - [实例](README_CHINESE.md#实例)
 - [常见问题](README_CHINESE.md#常见问题)

## Availability
目前GitHub提供IsoSeq3 预售版本的下载
[releases](https://github.com/PacificBiosciences/IsoSeq3/releases)。
亦可以使用[bioconda](https://bioconda.github.io/)进行安装:

```
    conda install isoseq3
```

注意⚠️：GitHub和Bioconda上的版本均不是正式release。

注意⚠️：GitHub和Bioconda上的版本只能在Linux操作系统使用。

注意⚠️：GitHub和Bioconda上的程序并非ISO compliant，只可用于研究，不用于医学诊断和治疗。

注意⚠️：PacBio官方客户服务只支持正式发售的，稳定版SMRT Link中的Iso-Seq协议，不支持
GitHub和Bioconda上的非正式release程序。

注意⚠️：请通过GitHub Issues咨询您遇到的预售程序的问题。

注意⚠️：GitHub和Bioconda上的版本要求电脑CPU支持**SSE4.1**，所有2008 （Penryn）之后的CPUs都支持SSE4.1


## SMRTbell 设计

针对IsoSeq样本制库，PacBio支持三种不同形式的SMRTbell设计。
所有的设计都使用非对称的（asymmetric primers）标记转录子，而聚腺苷酸链（PolyA）不是必须的。
如需Barcode样本，barcodes序列需链接到3'端 primer。

<img width="600px" src="doc/img/isoseq3-barcoding.png"/>

## 流程

<img width="1000px" src="doc/img/isoseq3-workflow.png"/>

### 输入
对每个被分析的PacBio SMRTCell，须提供以下文件：

    <movie>.subreads.bam
    <movie>.subreads.bam.pbi


### CCS 单分子环状一致性序列

调用[*ccs*](https://github.com/PacificBiosciences/unanimity) 对每个ZMW进行
一致性分析（consensus calling）以得到代表该ZMW的单分子环状一致性序列CCS。进行CCS
一致性分析的ZMW必须包含至少一个full pass，即每个primer都至少被观察到一次。
此外，IsoSeq的CCS调用无需（ccs polish）进行打磨纠错.
例子：

```
ccs movie.subreads.bam ccs.bam --no-polish --num-passes 1
```

### Primer removal 以及 demultiplexing
IsoSeq3调用[*lima*](https://github.com/pacificbiosciences/barcoding)以
去除primers，检测barcodes，并把不同barcode的ccs序列分开放入不同输出文件
（demultiplexing）。

注意⚠️：针对IsoSeq的的lima调用需使用`--isoseq` 模式。

注意⚠️：如果样本库是含barcode的池化，*lima*将去除primer并进行demultiplexing。否则
*lima*将仅仅去除primer。

注意⚠️：IsoSeq3要求输入的primer文件（或primer+barcode）文件必须满足一定的规则，请
查阅[FAQ](https://github.com/pacificbiosciences/barcoding#how-can-i-demultiplex-isoseq-data)
以确保您提供的输入文件正确.

    lima ccs.bam barcoded_primers.fasta demux.ccs.bam --isoseq --no-pbi

PacBio官方推荐 Clontech SMARTer cDNA library prep的样本制库协议。
下面例子为这种官方推荐制库协议在IsoSeq3中所对应的`primer.fasta`。

    >primer_5p
    AAGCAGTGGTATCAACGCAGAGTACATGGG
    >primer_3p
    GTACTCTGCGTTGATACCACTGCTT

下面例子为16bp barcode + Clontech primer的样本库所对应的`primer.fasta`。

    >primer_5p
    AAGCAGTGGTATCAACGCAGAGTACATGGGG
    >brain_3p
    CGCACTCTGATATGTGGTACTCTGCGTTGATACCACTGCTT
    >liver_3p
    CTCACAGTCTGTGTGTGTACTCTGCGTTGATACCACTGCTT

*lima* 将 根据输入的`primer.fasta`，侦测并去除 包含错误的primer组合的ccs 或 方向错误的ccs。

接下来，对每一个*lima*输出的BAM文件执行以下步骤。

### 聚类 （Clustering） 和 打磨纠错 （Polishing）
*IsoSeq3* 以 子命令 的形式，提供所有IsoSeq所需执行任务。

    $ isoseq3

    isoseq3 - De Novo Transcript Reconstruction

    Tools:
        cluster   - Cluster CCS reads to transcripts   （聚类）
        polish    - Polish the clustering output       （打磨纠错）
        summarize - Create a barcode overview CSV file （总结）

    Examples:
        isoseq3 cluster movie.consensusreadset.xml unpolished.bam
        isoseq3 polish unpolished.bam movie.subreadset.xml polished.bam
        isoseq3 summarize polished.bam summary.csv

#### 聚类 （Clustering）和 转录组数据清理 （transcript clean up）
*IsoSeq3* 使用 单聚类 技术。
由于算法的自身特性，这个聚类任务无法被高效的分解成并行计算的多个子任务，因此我们建议
在执行`isoseq3 cluster`时，不将其拆分成子任务，而是使用较多的CPU内核以加快运行速度。

*isoseq3 cluster*包含如下功能：

 - 去除聚腺苷酸链 [Trimming of polyA tails](https://github.com/PacificBiosciences/trim_isoseq_polyA)
 - 去除人为并联读子 [concatmer identification and removal](https://github.com/jeffdaily/parasail)
 - 分层迭代聚类 [Clustering using hierarchical n*log(n) alignment](https://github.com/lh3/minimap2) and iterative cluster merging
 - 生成粗粒度多分子一致性序列 [Unpolished POA sequence generation](https://github.com/rvaser/spoa)

##### 输入
*cluster* 的输入文件必须是*lima* demultiplex 生成CCS文件:

 - `<demux.ccs.bam>` or `<demux.ccs.consensusreadset.xml>`

##### 输出
*cluster* 的输出文件包括：

 - `<prefix>.bam`      <- Unpolished consensus isoforms bam
 - `<prefix>.flnc.bam` <- Full-Length Non-Chimeric reads
 - `<prefix>.fasta`    <- Unpolished consensus isoforms fasta
 - `<prefix>.bam.pbi`  <- Only generated with `--pbi`
 - `<prefix>.transcriptset.xml`    <- Only relevant for pbsmrtpipe
 - `<prefix>.consensusreadset.xml` <- Only relevant for pbsmrtpipe

例子:

```
isoseq3 cluster demux.P5--P3.bam unpolished.bam --verbose
```

#### 打磨纠错（Polishing）
IsoSeq3使用 *isoseq3 polish* 命令进行打磨纠错得到isoform的多分子一致性序列。
*polish* 是可选任务，但是我们强烈推荐使用它。
*polish* 的算法和CCS的算法是一致的，但使用多个ZMW的数据进行纠错可以得到更高精度的一致性序列。
*polish* 的算法和在基因组组装（de-novo assemblies）中使用的打磨纠错算法相同。

*polish* 这个任务很容易被分解成大量可并行计算的子任务，具体如下：

  - 调用 `isoseq3 cluster --split-bam`将`unpolished.bam`分成多个小文件
  - 对每个生成的小文件调用`isoseq3 polish`进行打磨纠错，生成`polished.bam`
  - 合并所有生成的`polished.bam`文件

例子：

```
    isoseq3 cluster unpolished.bam unpolished.chunk.bam --split-bam 3
    for i in 1 2 3; do \
        isoseq3 polish unpolished.chunk.$i.bam <movie>.subreads.bam polished.$i.bam \
    done
    samtools merge polished.merged.bam polished.*.bam -c -p
```

##### 输入
*polish* 的输入文件:

 - `<unpolished>.bam` 或者 `<unpolished>.transcriptset.xml`
 - `<movie_name>.subreads.bam` 或者 `<movie_name>.subreadset.xml`

##### 输出
*polish* 的输出文件包含多分子打磨纠错后的isoforms：

 - `<prefix>.bam`     <- polished isoforms bam
 - `<prefix>.bam.pbi` <- Only generated with `--pbi`
 - `<prefix>.transcriptset.xml` <- Only relevant for pbsmrtpipe
 - `<prefix>.hq.fasta.gz` with predicted accuracy ≥ 0.99 <- 高质量 全长 转录异构体 fastq
 - `<prefix>.lq.fasta.gz` with predicted accuracy < 0.99 <- 低质量 全长 转录异构体 fastq
 - `<prefix>.hq.fastq.gz` with predicted accuracy ≥ 0.99 <- 高质量 全长 转录异构体 fasta
 - `<prefix>.lq.fastq.gz` with predicted accuracy < 0.99 <- 低质量 全长 转录异构体 fasta

例子：

```
isoseq3 polish unpolished.bam m54020_171110_2301211.subreads.bam polished.bam
```

## 安装

 - *ccs*: 安装官方 [SMRT Link](https://www.pacb.com/support/software-downloads/) 或者 从源代码编译 [unanimity](https://github.com/PacificBiosciences/unanimity)
 - *lima*: 下载预编译的执行程序 [barcoding](https://github.com/pacificbiosciences/barcoding) 
 或者 `conda install lima`
 - *isoseq3*: 下载预编译的执行程序 [releases](https://github.com/PacificBiosciences/IsoSeq3/releases) 或者 `conda install isoseq3`
 
将包含`ccs` `lima` `isoseq3`程序的文件夹加入`PATH`:

```
export PATH=$PATH:<path_to_binaries>
```

## 实例
下面描述了从subreads开始到生成打磨纠错过的转录异构体（polished isoforms）结束的命令行流程。

注意⚠️： `wget` `time`等辅助程序依赖于电脑操作系统OS，`isoseq3`不提供


    $ wget https://downloads.pacbcloud.com/public/dataset/RC0_1cell_2017/m54086_170204_081430.subreads.bam
    $ wget https://downloads.pacbcloud.com/public/dataset/RC0_1cell_2017/m54086_170204_081430.subreads.bam.pbi

    $ ccs --version
    ccs 3.0.0 (commit f9f505c)

    $ time ccs m54086_170204_081430.subreads.bam m54086_170204_081430.ccs.bam \
               --noPolish --minPasses 1

    real    50m43.090s
    user    3531m35.620s
    sys     24m36.884s

    $ cat primers.fasta
    >primer_5p
    AAGCAGTGGTATCAACGCAGAGTACATGGGG
    >primer_3p
    AAGCAGTGGTATCAACGCAGAGTAC

    $ lima --version
    lima 1.6.1 (commit v1.6.1-1-g77bd658)

    $ time lima m54086_170204_081430.ccs.bam primers.fasta demux.bam \
                --isoseq --no-pbi --dump-clips

    real    0m6.543s
    user    0m51.170s

    $ ls demux*
    demux.json  demux.lima.counts  demux.lima.report  demux.lima.summary  demux.primer_5p--primer_3p.bam  demux.primer_5p--primer_3p.subreadset.xml

    $ time isoseq3 cluster demux.primer_5p--primer_3p.bam unpolished.bam --verbose
    Read BAM                 : (200740) 8s 313ms
    India                    : (197869) 9s 204ms
    Save flnc file           : 35s 366ms
    Convert to reads         : 36s 967ms
    Sort Reads               : 69ms 756us
    Aligning Linear          : 42s 620ms
    Read to clusters         : 7s 506ms
    Aligning Linear          : 37s 595ms
    Merge by mapping         : 37s 645ms
    Consensus                : 1m 47s
    Merge by mapping         : 8s 861ms
    Consensus                : 12s 633ms
    Write output             : 3s 265ms
    Complete run time        : 5m 12s

    real    5m12.888s
    user    58m35.243s

    $ ls unpolished*
    unpolished.bam  unpolished.bam.pbi  unpolished.cluster  unpolished.fasta  unpolished.flnc.bam  unpolished.flnc.bam.pbi  unpolished.flnc.consensusreadset.xml  unpolished.transcriptset.xml

    $ time isoseq3 polish unpolished.bam m54086_170204_081430.subreads.bam polished.bam --verbose
    14561

    real    60m37.564s
    user    2832m8.382s
    $ ls polished*
    polished.bam  polished.bam.pbi  polished.hq.fasta.gz  polished.hq.fastq.gz  polished.lq.fasta.gz  polished.lq.fastq.gz  polished.transcriptset.xml

如果您有多个SMRT Cells，您可以在调用`isoseq3 cluster`时使用`--split-bam`选项，将生成
的`unpolished isoforms bam`文件分成若干小文件。
调用 `isoseq3 polish` 对每个小文件进行打磨纠错，得到`polished isoforms bam`文件，并将它们合并生成最终结果。

注意⚠️：针对不同小文件的 `isoseq3 polish` 任务可以并行处理，


例如，下面将polish任务分成了24小块:

   - 例子中的`sample.subreadset.xml`包含所有SMRT Cells。
   - 例子中的`isoseq3 polish`任务都可以并行处理。

    $ isoseq3 cluster demux.primer_5p--primer_3p.bam unpolished.bam --split-bam 24
    $ isoseq3 polish unpolished.0.bam sample.subreadset.xml polished.0.bam
    $ isoseq3 polish unpolished.1.bam sample.subreadset.xml polished.1.bam
    $ ...
    

## 常见问题
### 为何我们开发IsoSeq3，并推荐用户使用IsoSeq3？

随着 PacBio Sequel 系统测序数据通量不断的增加，我们需要可快速处理百万级CCS数据的IsoSeq算法。
此算法必须 可扩展，快速高效，并且保持 高敏感性 和 高特异性。
PacBio研发部门对IsoSeq3进行了大量测试，结果显示IsoSeq3运行速度比IsoSeq1和IsoSeq2快十几到几十倍。
同时[SQANTI](https://bitbucket.org/ConesaLab/sqanti)的评估显示IsoSeq3生成更多数量完美符合参考转录组的数据。
此外，IsoSeq3非常容易安装，只Linux操作系统，除此以外无其他依赖关系。


### 为何IsoSeq3生成数量较少的转录异构体？

我们观察到IsoSeq3生成 数量稍少，但质量更高的转录组数据（polished isoforms）。
主要原因是在*lima* `demultiplexing`步骤过滤更多低质量的转录组数据。
*Isoseq1/2 classify*的过滤标准较为宽松，导致一些质量较差或有缺陷的数据被保留成Full-Length Non-Chimeric reads，并进入聚类和打磨纠错任务。
IsoSeq3使用`lima`进行`demultiplexing`，`lima`更精确得检测并过滤低质量和有缺陷数据。
比如`lima`能非常精准的检测和剔除含错误标签的ccs，比如两个5‘ 或者两个3’或者TSO（Template Switching)
缺陷数据。
`lima`识别 含有正确5‘ primer 和 3’ primer，并且有合适PolyA的CCS，并进行处理
以生成全长读子（Full-Length Non-Chimeric reads）。

### 为何我找不到*classify* 步骤

*Classify* 现在通过调用 PacBio标准 demultiplexing 工具 *lima* 通过`--isoseq`模式提供.

注意⚠️：*Lima* 并不去除PolyA聚腺苷酸链，也不检测人工联合读子（concatemer）。
如何去除PolyA和联合读子 在下一个问题解释。


### 我如何得到FLNC reads（全长 无5‘ primer，无3‘ primer，无PolyA，过滤人工缺陷读子）全长读子?

聚类任务 `isoseq3 cluster` 第一步生成　`*.flnc.bam`文件，包含所有FLNC reads。
如果您只需要FLNC reads，在`*.flnc.bam`文件生成后，您可以终止`isoseq3 cluster`任务。


### 需要多久才能处理完所有我的数据？

目前`isoseq3`没有估计剩余时间的功能。运行时间会根据样本的类型不一样而变化，全转录组分析 和
靶向转录组分析 速度也不一样。我们的测试数据显示 ，如果使用 16 个CPU core，处理一个SMRT Cell，分析 全转录组样本 在几分钟内完成，而分析10kb 靶向转录组样本 需要几个小时。


### 聚类Clustering使用什么算法？

与IsoSeq1 以及 IsoSeq2不同，*IsoSeq3*不使用NP-Hard的最大团寻找算法（clique finding）；
而是使用分层对比的策略。该算法复杂度是`O(N*log(N))`。
感谢近年来的快速对比算法的快速发展，使得我们开发此新算法成为可能。

### *Cluster* 使用多少CCS读子 以 生成 未经打磨纠错的的转录组类的代表序列（unpolished cluster 
sequence representation）?

*Cluster* 最多使用 10 个CCS读子以生成 一个未经打磨纠错的的转录组类的代表序列（unpolished cluster sequence representation）.

### *Polish* 使用多少subreads来进行打磨纠错（Polish）？

*Polish* 最多使用 60 个subreads读子以转录异构体（isoform）进行对多分子打磨纠错(polish the cluster consensus)，以得到多分子一致性序列。

### 划为同一个类的两个读子必须满足什么条件？

如果任两个读子满足以下所有条件，*IsoSeq3* 认为它们来自同一个转录子：

<img width="1000px" src="doc/img/isoseq3-similar-transcripts.png"/>

注意⚠️：5‘差异必须少于100bp

注意⚠️：3’差异必须少于30bp

注意⚠️：此外，内部任何连续gap（包括mismatch/indels）必须少于10bp

注意⚠️：内部gap只有10bp长度，没有数量限制。


### BAM tags详述

*IsoSeq3* 使用以下BAM tags：

 - `ib` Barcode 综述: 以分号分隔的三元组列表，每个三元组个体都包含 以逗号分隔的 两个barcode的索引 以及 一个 ZMNW数目。例如: `0,1,20;0,3,5`表示有两个三元组个体，第一个三元组是`0,1,20`，表示样本barcode的索引分别是`0`和`1`，支持此isoform的ZMW共有`20`个。第二个三组是`0,3,5`表示样本barcode索引分别是`0`和`3`，支持此isoform的ZMW共有`5`个。
 - `im` 支持该转录异构体（isoform）的ZMW数目
 - `is` 支持该转录异构体（isoform）的ZMW名称
 - `iz` 打磨纠错（Polish）中使用的最大数量的subreads
 - `rq` 预测 打磨纠错后的多分子一致性序列（Polished isoform）的准确度

 最大 Quality values 为 `93`.

## DISCLAIMER

THIS WEBSITE AND CONTENT AND ALL SITE-RELATED SERVICES, INCLUDING ANY DATA, ARE PROVIDED "AS IS," WITH ALL FAULTS, WITH NO REPRESENTATIONS OR WARRANTIES OF ANY KIND, EITHER EXPRESS OR IMPLIED, INCLUDING, BUT NOT LIMITED TO, ANY WARRANTIES OF MERCHANTABILITY, SATISFACTORY QUALITY, NON-INFRINGEMENT OR FITNESS FOR A PARTICULAR PURPOSE. YOU ASSUME TOTAL RESPONSIBILITY AND RISK FOR YOUR USE OF THIS SITE, ALL SITE-RELATED SERVICES, AND ANY THIRD PARTY WEBSITES OR APPLICATIONS. NO ORAL OR WRITTEN INFORMATION OR ADVICE SHALL CREATE A WARRANTY OF ANY KIND. ANY REFERENCES TO SPECIFIC PRODUCTS OR SERVICES ON THE WEBSITES DO NOT CONSTITUTE OR IMPLY A RECOMMENDATION OR ENDORSEMENT BY PACIFIC BIOSCIENCES.
