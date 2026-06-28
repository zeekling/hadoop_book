## Origami_ICPP25.pdf (PDF, 970.6 KB)

- **Title**: Origami: Efficient ML-Driven Metadata Load Balancing for Distributed File Systems
- **Created**: D:20250810132148Z
- **Modified**: D:20250810132148Z
- **Pages**: 11
- **Format**: PDF
- **Word Count**: 7211

- Words: 7211 | Chars: 50,064 | Pages: 11

> **Note:** Content was truncated (50,064 of 63,091 chars returned). Use maxChars for a higher limit.

### Content

Origami: Efficient ML-Driven Metadata Load Balancing for
Distributed File Systems
Yiduo Wang
China Telecom Cloud Computing
Research Institute
Beijing, Beijing, China
wangyd22@chinatelecom.cn
Wenda Tang
China Telecom Cloud Computing
Research Institute
Beijing, Beijing, China
tangwd1@chinatelecom.cn
Linghang Meng
China Telecom Cloud Computing
Research Institute
Beijing, Beijing, China
menglh1@chinatelecom.cn
Liang Li
China Telecom Cloud Computing
Research Institute
Beijing, Beijing, China
lil225@chinatelecom.cn
Jie Wu
China Telecom Cloud Computing
Research Institute
Beijing, Beijing, China
wujie@chinatelecom.cn
Abstract
Modern distributed file systems (DFSs) rely on metadata server clus-
ters to manage large-scale files and achieve scalability. However,
the hierarchical namespace structure and dynamic user workloads
pose severe challenges for efficient metadata partitioning and load
balancing. Existing approaches primarily focus on identifying and
redistributing hot metadata to address imbalances. While these
load-balancing strategies offer potential benefits, they often reduce
metadata locality, ultimately failing to improve the end-to-end job
completion time—a key metric prioritized by users. Although re-
cent research reveals that learning-based approaches are effective
in predicting hotspots, they have been shown to be less effective in
improving metadata performance. We revisit metadata load balanc-
ing strategies and propose a learning-based metadata load balance
frameworkOrigami, which focuses on minimizing end-to-end job
completion time rather than equalizing loads. Origami first decom-
poses the overhead of metadata operations and assesses the impact
of migration decisions on user requests, allowing us to compute the
benefits of migration decisions for job completion time when future
requests are known. Subsequently, Origami propose the Meta-OPT
algorithm to determine near-optimal migration decisions. Finally,
we implemented OrigamiFS, on which we collected statistical data
to train and validate ML-models capable of predicting migration
benefits. By predicting the benefits of migration decisions and em-
ploying Meta-OPT to quickly explore nearly optimal migration
decisions, Origami makes a better trade-off between load balancing
and namespace locality. Our evaluation shows that compared to
state-of-the-art methods, Origami increases aggregated metadata
throughput by 1.12-2.51×across three real-world workloads, and
enhances end-to-end throughput by 1.11-2.02×.
This work is licensed under a Creative Commons Attribution 4.0 International License.
ICPP ’25, San Diego, CA, USA
©2025 Copyright held by the owner/author(s).
ACM ISBN 979-8-4007-2074-1/25/09
https://doi.org/10.1145/3754598.3754617
CCS Concepts
•Software and its engineering→Distributed systems organiz-
ing principles;•Information systems→Distributed storage;
•Social and professional topics→File systems management.
Keywords
Distributed file system, Metadata management, Machine learning
ACM Reference Format:
Yiduo Wang, Wenda Tang, Linghang Meng, Liang Li, and Jie Wu. 2025.
Origami: Efficient ML-Driven Metadata Load Balancing for Distributed File
Systems. In54th International Conference on Parallel Processing (ICPP ’25),
September 08–11, 2025, San Diego, CA, USA.ACM, New York, NY, USA,
11 pages. https://doi.org/10.1145/3754598.3754617
1  Introduction
Modern data centers typically deploy large-scale distributed file
systems (DFSs) to manage large amounts of files [9,19,24,30].
To enable efficient file indexing and processing, DFSs often rely
on dedicated metadata server (MDS) clusters to store metadata
and accelerate metadata operations (e.g., file lookups and direc-
tory creation) [11,25,43]. As the scale of distributed file systems
has grown to encompass hundreds of billions of files, distributing
metadata across multiple MDSs for parallel processing has become
a common practice. This approach, widely adopted by internet
and cloud service providers, is crucial to enhancing scalability and
performance [7, 21, 22, 32].
File workloads in modern datacenters have exhibited increas-
ingly metadata-intensive characteristics: over 90% of files and I/O
requests are smaller than 1MB, resulting in a continuously rising
proportion of metadata operations [26,40]. Currently, metadata
has emerged as the primary bottleneck in DFS, which requires
careful partitioning to deliver high-performance metadata services.
Unlike storage systems with flat namespaces, such as key-value
stores or object stores, balancing the metadata load in hierarchical
namespaces(i.e., a directory tree structured as a directed acyclic
graph) should be more cautious. First, real-world workloads are
diverse and dynamic, often leading to hotspots within hierarchi-
cal namespaces, which necessitate timely migration of subtrees.
Second, partitioning or migrating metadata across multiple nodes

ICPP ’25, September 08–11, 2025, San Diego, CA, USAYiduo Wang, Wenda Tang, Linghang Meng, Liang Li, and Jie Wu
introduces additional overhead for metadata operations: (1) path
resolution requires multiple RPCs (remote process calls) to traverse
the entire path when nodes along a file path are stored on differ-
ent servers [26,29]; (2) metadata operations involving concurrent
access across multiple nodes (e.g., directory reads or namespace
structure mutations) also incur higher costs [10, 40].
Prior work has explored metadata partitioning from various
perspectives. CephFS [44] and HopsFS [29] adopt coarse-grained
partitioning to maintain namespace locality, while InfiniFS [26] and
CFS [40] focus on minimizing the overhead of fine-grained partition-
ing. Moreover, Mantle [34] introduces programmability to improve
metadata load balancing, and Lunule [39] leverages both temporal
and spatial locality to enhance load balancing. These studies have
primarily focused on thepopularity[44] of directories or files (i.e.,
their load levels) and enabling migration between MDSs to achieve
balance. More recently, some works have introduced machine learn-
ing (ML) to predict future popularity and improve metadata load
balancing [8,41,42]. In general, existing approaches rely on heuris-
tic algorithms or ML-based methods to predict hotspots, and then
use this information to guide metadata partitioning.
However, load balancing in hierarchical file system namespaces
is significantly more complex than in flat namespaces. In particular,
simply predicting or classifying the metadata as “hot” or “cold” is in-
sufficient to achieve optimal load balancing. We argue that ML meth-
ods should be leveraged to identify efficient migration decisions in
DFS, but this requires solving the following unique challenges: (1)
trade-offs between the benefits and costs of load balancing.Metadata
migration can disrupt the namespace locality, introducing addi-
tional overhead with complex and operation-dependent impacts
that vary significantly between different operations and sequences.
(2)finding the optimal balancing strategy is difficult.Modern DFSs
typically have massive metadata, making it difficult to quickly find
the optimal migration strategy, and aggressive migration strategy
can even significantly reduce the efficiency of metadata services.
(3) Finally,existing DFSs are not designed to be ML-native, which
hinders the collection of relevant feature data and the execution of
the learned strategies, further complicating balanace solutions.
Based on these observations, this paper advocates thatevenly
distributing metadata load should NOT be the primary optimization
objective. Instead, we propose a novel framework,Origami, for
training efficient ML-driven metadata load balancing strategies.
Origami shifts the focus from precise metadata load prediction to
optimizing MDS cluster efficiency and minimizing end-to-end job
completion time. The main contributions of this paper consist of:
•Estimating metadata operation execution time: We pro-
pose an approach to predict the completion time for metadata
operations and job, based on a given namespace partition
and user requests sequence, which provides a key metric for
evaluating balancing strategies’s benefit and costs.
•Finding near-optimal migration decisions: Inspired by
the classic Bélády’s algorithm [45], we designMeta-OPT, a
mechanism designed to calculate end-to-end migration ben-
efits, and then identify near-optimal migration decisions for
a given sequence of metadata operations. Meta-OPT sub-
sequently guides ML-training in identifying near-optimal
migration strategies.
Clients
DistributedFileSystem
Data
Metadata
MDS
-
0
usr
create(“/usr/bin/dir/foo”)
②pathresolution
③pathresolution
&filecreation
MDS
-
1
bin
share
dir
MDS
-
2
etc
sbin
/
bar
foo
tmp
①pathresolution
Figure 1: Modern DFS architecture and user workflow.
•
Training ML methods to optimize metadata: We design a
framework, Origami, with a ML-native distributed metadata
serviceOrigamiFS, that automates feature collection from
user workload, applies ML-based algorithms, and evaluates
their effects on end-to-end job performance.
We evaluated Origami with 5 MDSs and found that Origami
achieves a better trade-off between load balance and namespace lo-
cality. Specifically, Origami increases metadata throughput to 3.86×
that of a single MDS while incurring only a 3.5% increase in for-
warded requests. Compared to subtree- and hash-based partitioning
approaches, Origami achieves the best overall performance, boost-
ing the cluster’s aggregate metadata throughput and end-to-end
file throughput by factors of 1.12-2.51×and 1.11-2.02×, respectively,
in 3 real-world workloads.
2  Background and Motivation
2.1  Distributed Metadata Management
GFS [9] and HDFS [11] introduced a key technique in the evolu-
tion of DFS: decoupling metadata from file data. By centralizing
metadata management on a dedicated MDS and distributing file
data evenly across numerous data servers, DFSs achieve improved
scalability. However, as the average file size has decreased from
GBs to MBs and the number of files managed by a single DFS has
grown to hundreds of billions, a single metadata server is no longer
sufficient to meet the demands of processing speed and storage ca-
pacity [1,3,18,35]. Recent studies have shown that workloads in the
cloud exhibit significant metadata-intensive characteristics, with
metadata operations often accounting for more than two thirds of
the workload in most cases, and in some scenarios, even exceeding
90% [40], which has become a major bottleneck in DFS.
To overcome these limitations, modern DFSs partition the names-
pace into multiple metadata shards and distribute them across
MDSs, enabling parallel processing and further enhancing scal-
ability. Figure 1 illustrates the architecture of modern DFSs, con-
sisting of three components: a metadata cluster for namespace
management, a data cluster for file storage, and clients that initiat-
ing user requests. In this setup, clients must access metadata before

Origami: Efficient ML-Driven Metadata Load Balancing for Distributed File SystemsICPP ’25, September 08–11, 2025, San Diego, CA, USA
0.0
0.5
1.0
1.5
2.0
2.5
051015202530
Throughput of single MDS
Normalized Throughput
Time(min)
AM1M2M3M4M5
(a) Throughput
 0
 25
 50
 75
15
57%
Execution Time (min)
# of MDSs
(b)  Job  Completion
Time
Figure 2: (a) Normalized metadata throughput across 5 MDSs
(M1-M5: Per MDS, A: Aggregated) (b) Job completion time
with 1 MDS vs. 5 MDSs.
retrieving file data. As shown in the upper half of this figure, a file
creation (e.g., “/foo” under “/usr/bin/dir/”) requires sequentialpath
resolution(i.e., traverse the metadata along the path) to verify direc-
tory existence and permissions before execute metadata mutation.
This design provides scalability, POSIX [38] compliance, and user
programs compatibility but increases metadata overhead due to
additional remote procedure calls (RPCs) [2, 13, 46].
2.2  Even Partitioning Considered Harmful
The hierarchical namespace and path resolution process presents
significant challenges for metadata load balancing. Unlike many
storage systems with flat namespaces (e.g., object stores [5]), evenly
distribute metadata in DFS can result in notable performance degra-
dation. To analyze the potential performance degradation caused
by balancing, we deployed a CephFS cluster with 5-MDSs , an open
source and widely used DFS, and ran 50 clients to saturate MDSs.
In alignment with prior studies, we replayed a web access work-
load [4], disabled the data path, and evenly distributed metadata per
directory via the built-in CephFS function [6,39]. Figure 2 compares
the performance of a single MDS set-up with a configuration of
5-MDSs. As shown in Figure 2a, the throughput of each individual
MDS (5 solid lines) in the 5-MDSs configuration is significantly
lower than that of a single MDS (red dashed line). Note that this in-
efficiency is not caused by an insufficient client request load; rather,
it is due to the additional execution overhead, which limits the
processing capacity of each MDS (see details in § 5.5). Interestingly,
even after adding 4 MDSs, the aggregated metadata throughput
(purple dashed line) increased by only about 1.4 times. Meanwhile,
as shown in Figure 2b, the corresponding job completion time was
reduced by merely 57%.
This inefficiency is primarily due to the resolution and process-
ing overhead associated with load balancing, as described in Sec-
tion 2.1. This suggests that we must approach load balancing with
caution, focusing not just on evenly distributing the load, but also
on minimizing the associated overhead. It should be noted that our
experiments incorporated metadata caching and other optimization
techniques to mitigate these issues, but significant inefficiencies
persist (see details in § 5.4).
2.3  Limitations of Metadata Partitioning
Recent data centers and clouds statistics reveal that metadata has
surpassed data, becoming the main bottleneck for scaling DFS [18,
40]. To address that, modern DFSs have designed various metadata
partitioning strategies over the past decade. However, mainstream
methods still struggle to effectively balance locality and scalability.
First,Coarse-grained partitioningis hard to scale. For example,
CephFS’s dynamic subtree partitioning [44], a well-known coarse-
grained approach, aims to preserve the locality of the namespace
by migrating subtrees to other MDS only when load imbalance
occurs. Coarse-grained partitioning has been widely embraced by
numerous clouds and data centers due to its ability to maintain
locality [21,22,29]. Although this method effectively reduces the
additional metadata overhead, it often leads to metadata hotspots
under dynamic workloads, ultimately limiting the overall scalability
of the system [34, 39].
Second,Fine-grained partitioningincreases the metadata over-
head. Per-directory partitioning uses hash-based algorithms to
evenly distribute metadata across MDSs [23,25,30,36], achieving
better load balancing. However, disrupting namespace locality can
introduce additional overhead for metadata operations, mainly due
to RPC forwarding and distributed coordination [40]. Cloud ven-
dors report that this can lead to latency increasing almost linearly
with directory depth, rendering undesirable latency [7, 20, 26].
RecentML-based partitioningstrategies aim to predict the future
load and migrate files or folders by learning from historical load
data. Compared to heuristic strategies, ML-based strategies show
potential in flexibility and efficiency. However, existing learned
strategies overlook the hierarchical structure of metadata, suffer
from low prediction efficiency when handling dynamic loads, and
rely heavily on manual optimization, which constrains overall per-
formance improvements [8, 41, 42].
2.4  Challenges of ML-based Balancing
To address the metadata load balancing problem, our goal is to
develop an efficient ML-driven metadata load balancing mechanism,
building upon existing dynamic subtree and machine learning-
based strategies. To achieve this objective, several key challenges
must be addressed.
Challenges #1: Measuring appropriate metrics.Simply relying
on the number of inodes or the balance of QPS across different MDS
is not an appropriate metric, as these measures neither account for
the disruption of locality due to load balancing nor can they be
mapped to the completion times of user tasks.
Challenges #2: Finding effective migration decisions.The hi-
erarchical structure of the namespace poses significant challenges in
the search for optimal migration strategies. The file system names-
pace is not only deeply layered, exceeding ten levels, which hinders
rapid searches, but migration strategies between parent and child
nodes can also interfere with each other, and excessive migrations
can significantly affect system performance.
Challenges #3: Collecting statistics and validating models.
Existing distributed filesystems are not native to ML, making it
difficult to collect effective training data and validate the efficacy
of migration decisions.

ICPP ’25, September 08–11, 2025, San Diego, CA, USAYiduo Wang, Wenda Tang, Linghang Meng, Liang Li, and Jie Wu
3  Potential Benefits of Balancing
The observations and analysis mentioned above suggest that, in
DFS,even partitioning is NOT suitable for metadata load balancing.
To effectively manage metadata, it is crucial to prioritize reduc-
ing end-to-endJob Completion Time(JCT) during migration and
to train models aimed at minimizing JCT. Determining the JCT
prior to job execution is impractical. Therefore, we relax this goal
by computing the JCT for a given metadata request and names-
pace partition, which subsequently guides the learned model to
predict the migration benefits. Specifically, our approach, inspired
by the Bélády’s algorithm and learned cache [45], addresses two
key challenges:
Address Challenge #1: Measure the completion time of the
user request.We leverage therequest completion time(RCT) and
the corresponding migration benefits, both derived from a given
metadata access sequence and namespace partition. Specifically,
we decompose the components of RCT and compute its value for a
given namespace partition (§ 3.1).
Address Challenge #2: Find near-optimal decisions quickly.
Using the calculated RCT, we proceed to determine nearly optimal
metadata migration choices to reduce the total job completion time,
which then informs our partitioning strategies developed(§ 3.2).
3.1  Measure RCT Instead of Balance
Dive into metadata overhead.The main challenge in computing
RCT is that metadata overhead depends on both network/load con-
ditions and the hierarchical namespace structure. Under different
partitions, the same request may access varying numbers of MDSs.
Fortunately, metadata operations exhibit fixed patterns, enabling
us to decompose the overhead. Specifically, for a metadata request
with a path length of푘that is distributed among푚distinct meta-
data partitions, we assume that the processing overhead (e.g, time
of resolving path, creating file and updating parent attributes) of
metadata is푇
푚푒푡푎
, the queuing time on each partition is푄, and
the network request access time is푅푇푇. Consequently, RCT can be
expressed using the following formula:
푅퐶푇=푇
푚푒푡푎
+푚·푅푇푇+
푚
∑︁
푖=1
푄
푖
(1)
For metadata requests, both the queueing time and the network
request time can be computed based on the average network latency
and the machine load level, whereas the composition of푇
푚푒푡푎
varies
depending on the type of operation. In addition, apart from the
path resolution time푇
푝푎푡ℎ
,푇
푚푒푡푎
is also influenced by the metadata
partitioning strategy, which makes it difficult to predict.
Calculate additional overhead.Fortunately, we have found that
primary metadata requests can be categorized into three types, with
their execution times discussed separately. Specifically,푇
푚푒푡푎
con-
sists of a fixed baseline cost and additional variable overhead that
requires case-by-case analysis. The baseline cost includes (m + k)
inode reads, where m additional fake-inodes are stored to preserve
migration information. We categorize metadata into three types:
list directory (lsdir), namespace mutation (e.g.create,rmdir, short
in ns-m), and others unaffected operations. Forlsdir, migrating
the sub-files/directories to푖other MDSs introduces an additional
푖·푅푇푇. Furthermore, distributing the parent directory and the target
Algorithm 1:Meta-OPT Algorithm
Input  :A sequence of metadata operations푁;
A list of MDS
®
푀={푚
푖
}; A thresholdΔ;
Output:A list of migration decisions
®
퐷
1repeat
2푇←퐽퐶푇(푁,
®
푀);
3푚푎푥_푏푒푛푒푓푖푡←0;푏푒푠푡_푑푒푐푖푠푖표푛←∅;
4foreach푚
푖
∈
®
푀do
5foreachsubtree푠∈푚
푖
do
6foreach푚
푘
∈
®
푀\{푚
푖
}do
7
®
푀
′
=
®
푀.푚푖푔푟푎푡푒(푠,푚
푖
,푚
푘
);
8푇
′
=퐽퐶푇(푁,
®
푀
′
);
9if푇
′
<푇&(푚
′
푘
.푟푐푡−푚
′
푖
.푟푐푡)<Δthen
10푏푒푛푒푓푖푡←푇−푇
′
;
11if푏푒푛푒푓푖푡>푚푎푥_푏푒푛푒푓푖푡then
12푚푎푥_푏푒푛푒푓푖푡←푏푒푛푒푓푖푡;
13푏푒푠푡_푑푒푐푖푠푖표푛← (푠,푚
푖
,푚
푘
);
14
®
푀=
®
푀.푚푖푔푟푎푡푒(푏푒푠푡_푑푒푐푖푠푖표푛);
15
®
퐷.푝푢푠ℎ(푏푒푠푡_푑푒푐푖푠푖표푛);
16until푚푎푥_푏푒푛푒푓푖푡<푡ℎ푟푒푠ℎ표푙푑;
17return
®
퐷;
file/directory across MDSs incurs additional distributed coordina-
tion overhead for namespace mutation operations. Letting the read
time of the inode be푇
푖푛표푑푒
, the execution time be푇
푒푥푒푐
, and the
additional coordination time for distributed transactions be푇
푐표표푟
,I
mean the indicator function (1 if true and 0 if false), we can calculate
the execution time of metadata requests after migration:
푇
푚푒푡푎
=푇
푖푛표푑푒
·(푚+푘)+푇
푒푥푒푐
+








푅푇푇·푖,lsdir
푇
푐표표푟
·I(푖>0),ns-m
0,others
(2)
3.2  Estimating the Benefits of Migration
In this section, we first explore how to compute JCT and evaluate
the benefits of migration decisions. Based on this, we design a
Meta-OPT algorithm to identify near optimal migration strategies.
Estimate the job completion time.We compute the total cost and
RCT distribution across metadata partitions. Following Lunule [39],
we focus on high-load scenarios where the most loaded MDS ap-
proaches capacity, as load balancing benefits are most significant
under such conditions. Consequently, JCT can be approximated as
a bin-packing problem: MDSs serve as bins, with JCT estimated
by the largest bin’s capacity. Given the access sequence푁, we can
estimate the퐽퐶푇for the entire task in a way that, while not entirely
precise, remains simple and efficient: (1) calculate the total costs of
requests processed by each MDS (denoted as푚.푟푐푡) in the given
request sequence
1
. (2) sum the RCTs of requests processed by each
MDS, record the highest value as JCT.
1
Origami will estimating푇푞푢푒푢푒and푇푐표표푟via historical sampling data.

Origami: Efficient ML-Driven Metadata Load Balancing for Distributed File SystemsICPP ’25, September 08–11, 2025, San Diego, CA, USA
OrigamiFS
LabelGeneration
MetaOPT
①run
②collectstats
③generatelabels
&executedecisions
④continue
ModelTraining
...
MLmodels
ModelValidation
Online
Prediction
&Balancing
DataCollectorMigratorDataCollectorMigrator
traces
...
workload
...
Figure 3: The architecture of Origami for training efficient ML-based metadata balancing models.
Seek approximately optimal decisions.Here, we present the
Meta-OPT algorithm, designed to efficiently find an approximately
optimal migration decision given a known future metadata opera-
tions sequence푁and the current MDS state
®
푀. Details are shown
in Algorithm 1. The algorithm iteratively finds a list of migration
decisions that maximize the benefit and minimize the overall com-
pletion time푇of future metadata operations
®
푁(lines 2-3). This
process continues until the benefit drops below a predefined thresh-
old (line 16). Specifically, it traverses all subtrees within each MDS
(lines 4-5) and calculates the completion time푇
′
if the subtree is
migrated to another MDS (lines 6-8). If푇
′
is smaller than푇, indi-
cating a reduced overall completion time, the difference is recorded
as푏푒푛푒푓푖푡(lines 9-10). It should be noted that in order to prevent
new imbalance caused by the migration, we also require the imbal-
ance after migration is less than a specified thresholdΔin line 9.
During each iteration, the decision with the maximum푏푒푛푒푓푖푡is
selected (lines 11-13). After completion of the iteration, the selected
decision is executed, added to the migration decision list
®
퐷, and
the algorithm proceeds to the next iteration (lines 14-15).
It is important to note that, to find the optimal migration solution,
we need to enumerate all possible subsets of subtrees and identify
the set that yields the maximum benefit. However, exhaustively
evaluating all possible combinations of subtrees is computation-
ally infeasible. Therefore, Algorithm 1 adopts a greedy method. It
makes a sequence of local decisions, each time selecting the subtree
whose migration offers the greatest immediate benefit. Under this
approach, once a subtree푠is selected and migrated, any subtrees
nested within푠are no longer considered for migration. If there
exists a set of disjoint subtrees푘
1
,푘
2
, . . .,푘
푁
within푠whose to-
tal migration benefit exceeds that of migrating푠as a whole, then
the solution provided by Algorithm 1 is suboptimal. Nevertheless,
we can prove that the gap between Algorithm 1 and the optimal
solution is less thanΔ, as shown in the following theorem.
Theorem 1.Let푏
0
denote the benefit of migrating a subtree푠.
Assume that subtrees푘
1
,푘
2
, . . .,푘
푁
are푁disjoint subtrees nested
within subtree푠, and let푏
1
be the benefit of migrating푘
1
,푘
2
, . . .,푘
푁
.
Under the conditions specified in Algorithm 1, we have푏
0
−푏
1
>−Δ.
The proof of this theorem is provided in Appendix A. This the-
orem shows that even under suboptimal conditions, Algorithm 1
maintains performance within a controlled margin of error.
4  Toward Efficient ML Balancing
Address Challenge #3: Building a framework for training
efficient metadata balancing models.In this section, we initially
outline Origami, the system framework crafted for training efficient
learned metadata balancing models (§ 4.1). After that, we present the
architecture of OrigamiFS in (§ 4.2), the prototype metadata service
within Origami, and detail the complete workflow for training
metadata balancing models in (§ 4.3).
4.1  System Overview
Origami is supported by two key components: the Meta-OPT (§ 3.2)
algorithm and a lightweight distributed metadata service Origam-
iFS. Meta-OPT, an implementation of Algorithm 1, guides the ML
model in making migration decisions. OrigamiFS, implemented as
a prototype distributed metadata service, generates training fea-
tures and evaluates the effectiveness of models. As illustrated at
the bottom of Figure 3, we introduced two components for each
MDS to further support the Meta-OPT algorithm and ML models
in optimizing migration policies:
•Data Collector to dump the runtime namespace state.This
component outputs the metadata partition and attribute statistics.
Unlike snapshots, it focuses solely on feature data for ML training
(e.g., load in last epoch, depth,links) and metadata required for
Meta-OPT calculations. In addition, modern DFSs use directories
as the basic unit for load balancing, allowing us to omit file-level
metadata and significantly reduce the data collection overhead.
•Migrator to execute external migration decisions.Modern
DFS often integrate metadata load balancing as built-in logic or
pluggable functions [34]. To better support ML models, Migrator
enables external algorithms (e.g., Meta-OPT and ML models), to
provide migration decisions (e.g., path, source MDS, destination
MDS) in a pipeline manner.
4.2  OrigamiFS Architecture
We implemented OrigamiFS in the Go programming language,
which consists of the following parts:
Metadata Cluster.As Figure 4 shows, the metadata service within
OrigamiFS is composed of MDS units ranging from 0 to n. Each
MDS stores its individual inodes as key-value pairs in a local Peb-
blesDB [31]. Using the inode number of the parent directory com-
bined with the file name as an index, aligned with state-of-the-art
studies [12, 26, 40].

ICPP ’25, September 08–11, 2025, San Diego, CA, USAYiduo Wang, Wenda Tang, Linghang Meng, Liang Li, and Jie Wu
UserClient
UserClient
OrigamiFSSDK
near-rootcache
UserClients
MLModels
sendstatistics
...
Metadatalogic
PebblesDB
Data
Collector
Migrator
MDS-0
migrate
Metadata
Balancer
return
benefits
M-1M-n
Figure 4: The architecture of OrigamiFS.
In the initial state, OrigamiFS stores all metadata on the MDS
numbered 0. During each epoch, all MDSs send metadata statistics
to the ML model throughData Collectors. The ML model will send
the predicted benefits to MDS-0’s Metadata Balancer, which em-
ploys the same load monitoring and rebalancing trigger mechanism
as Lunule [39], but with a different rebalancing algorithm. Tradi-
tional migration algorithms generally require using bin-packing-
like methods to select subtrees based on load differences between
MDSs. In contrast, the rebalancing algorithm of OrigamiFS is much
more intuitive: MDS-0 simply greedily selects the subtree with the
highest benefit and uses theMigratorto migrate it to the lightly
loaded MDS, repeating this process until the migration benefits of
all subtrees fall below a specified threshold.
Clients.We designed a OrigamiFS SDK that enables clients to
convert file system calls into corresponding metadata operations
directed to the MDSs. Similar to the workflow illustrated in Figure 1,
the client first resolves the path recursively and then issues the
corresponding metadata operations. To address the well-known
near-root hotspot problem and reduce path parsing overhead, we
introduced a configurable near-root metadata cache, which stores
metadata entries whose depth is less than a predefined threshold.
Although the near-root metadata cache is a straightforward design,
it is highly effective: since near-root metadata typically constitutes
less than 1% of the entire file system [26], this approach substantially
mitigates the near-root hotspot issue while avoiding the significant
consistency overhead associated with cache synchronization or
lease management.
4.3  Origami Workflow
As shown in Figure 3, we train and validate efficient ML-based load
balancing models through the following process:
Label generation.We begin by collect the access traces from real-
world workloads, and replay the metadata operations on OrigamiFS
(①). After each epoch
2
, the Data Collector dumps metadata statis-
tics (②) from OrigamiFS. Next, Meta-OPT extracts features from
these statistics, performs normalization, calculates the migration
benefit for each metadata subtree, and uses these benefits as train-
ing labels step by step. Migration decisions with high estimated
benefits are then applied to rebalance OrigamiFS (③). The above
2
In the experiments of this paper, epoch is set to 10 seconds.
Table 1: Training features and the Gini importance rank
obtained after training with LightGBM.
TypeFeatureNormalizationGI Rank
Namespace
Structure
depth
by the max value
7
# sub-files1
# sub-dirs4
Metadata
History
# readby # total access
in last epoch
6
# write2
Derived
Feature
read-write ratio
raw
6
dir-file ratio2
process is repeated iteratively to progressively enrich the training
dataset (④).
Model training.After extracting feature and label data using
OrigamiFS and MetaOPT, we trained multiple models offline. As
Table 1 shows, for each directory, OrigamiFS outputs two types of
metadata statistics: (1)namespace structure statistics of this directory,
including directory depth, number of sub-files, and number of sub-
directories; (2)metadata access history in last epoch of this subtree, in-
cluding the total number of metadata read operations (e.g.,open(),
stat().) and metadata write operations (e.g.,create(),mkdir().)
in the previous epoch. Note that we refer to the access history of
the subtrees instead of that of the directory itself, since migration
is conducted on a subtree level. For namespace structure statistics,
we normalize using the maximum value of the corresponding items.
For metadata history, we normalize by taking the total count of
metadata operations from the previous epoch. Furthermore, we
incorporate additional features, such as the ratio of subdirectories
to subfiles and the proportion of write meta-operations to the total
number of meta-operations.
We developed a Python module and trained regression models
to predict benefits. We compared LightGBM [15], GBDT [16], and a
MLP [37] with 4 hidden layers. Interestingly, we found that although
there were slight differences in prediction accuracy among the three
models, the migration decisions produced when the predicted re-
sults were fed into the metaOPT algorithm were remarkably similar.
This occurred because each model succeeded in pinpointing sub-
trees with notably higher migration benefits, which is crucial for
the execution of the migration algorithm, while the migration algo-
rithm filtered out subtrees with lower benefits, thus minimizing the
effect of prediction inaccuracies. Consequently, we chose to imple-
ment the lightGBM model due to its minimal prediction overhead,
with 400 rounds of boosting and 32 leaves. Table 1 lists the specific
indicators used for the training, the corresponding normalization
methods, and the Gini importance [28] obtained after training.
Model validation.A key challenge of learning-based metadata bal-
ancing is that the accuracy of the model does not directly represent
the improvement in system performance. Fortunately, OrigamiFS
enables the online validation of different models. During runtime,
OrigamiFS asynchronously outputs metadata viaData Collector,
which the trained models use as feature input to predict migration
benefits. The high benefit migration decisions are then applied to

Origami: Efficient ML-Driven Metadata Load Balancing for Distributed File SystemsICPP ’25, September 08–11, 2025, San Diego, CA, USA
OrigamiFS through theMigrator, enabling online metadata rebal-
ancing. The above work allows us to evaluate the overall perfor-
mance optimization of metadata cluster directly, rather than relying
solely on accuracy metrics.
5  Evaluation
5.1  Experiment Setup
Hardware configurations.We validated Origami by developing
a prototype and implementing OrigamiFS with 5-MDSs, running
5 client nodes to saturate the capacity of MDS. The experiments
were conducted on a 10 node Kubernetes cluster, where each node
was equipped with 8 CPU cores, 64GB of RAM, and a 2TB NVMe
SSD for metadata processing and storage.
Baseline methods.We implemented state-of-the-art load balanc-
ing strategies in Origami for comparison, covering two categories:
•Hash-based partitioning: We reproduced two widely used hash-
based strategy in recent research and production systems: The
coarse-grained approach, akin toHopsFS[12], applies hashing
only to the upper levels of the namespace, which we label asC-
Hash. Conversely, the fine-grained approach hashes all directory,
which we denote asF-Hash, used inTectonicandInfiniFS[26,30].
•Popularity-based ML methods: We also reproduced the latest ML-
driven metadata load balancing method [42], using subtrees as
the basic granularity, and using the LightGBM model to predict
access popularity and guide load balancing, namedML-tree.
For hash-based methods, we partitioned the metadata before
conducting the evaluation. For both ML-tree and Origami, statistics
were collected after each epoch, using Lunule’s algorithm [39] to
trigger load rebalancing. To ensure a fair comparison, thenear-root
cachewas activated for all strategies. A standalone MDS was used
as the baseline for performance measurement.
Workload configurations.We selected the following 3 real-world
workloads that have been used in recent metadata studies:
(1)
Trace-RW: A large compilation task consisting of numerous
complex metadata operations [34].
(2)Trace-RO: A web application access trace, which only in-
cludes read-type operations, exhibits a significant skew and
extends to a considerable depth [39].
(3)Trace-WI: A write-intensive trace from a distributed file sys-
tem on the cloud, which we reproduced based on the char-
acteristics described in the paper [40].
For comparative analysis purposes, we use Trace-RW with only
the metadata function active, allowing us to evaluate and examine
the metadata performance and load balancing of various baselines
from § 5.2 to § 5.5. Finally, in § 5.6, we activated the data path and
compared all methods with three real-world workloads.
5.2  Overall Performance
We begin our analysis by evaluating the overall metadata perfor-
mance using Trace-RW. For each approach, we initiate 50 client
threads to fully utilize the metadata service and activate the load
balancing mechanism. Subsequently, we measure the average ag-
gregated metadata throughput post-rebalancing. Following this, we
rerun the workload with a single thread to compare the changes in
0
20K
40K
60K
80K
C-HashF-HashML-popOrigami
Single
MDS
Aggregated QPS
(a) Throughput
0
0.1
0.2
0.3
C-Hash
F-Hash
ML-tree
Origami
Single
MDS
Average Latency (ms)
(b) Latency
Figure 5: Aggregate throughput under high load and average
latency under single thread of different balancing methods.
latency under different methods, thereby evaluating the extent to
which each balancing approach disrupts namespace locality.
Aggregated throughout under high load.Figure 5a shows the
aggregate metadata throughput using different balancing strate-
gies. When a single MDS processes the metadata operations for
Trace-RW, OrigamiFS achieves a metadata throughput of 19.4k/s.
By distributing the metadata across multiple MDSs, C-Hash can
utilize multiple MDSs in parallel, increasing the throughput by
2.23×, which is consistent with our observations in § 2.2. However,
fine-grained hashing does not yield additional performance gains:
we found that F-Hash’s throughput decreased by 31.0% compared to
C-Hash. This is because, although F-Hash enables a more balanced
distribution of load across multiple MDSs, its significant disruption
to namespace locality results in overhead that outweighs the bene-
fits of load balancing in terms of metadata throughput. The results
for ML-tree fall between the two, achieving 1.89×the throughput
of a single MDS. We found that although ML-tree can predict hot
directories, it tends to overlooks the negative impact of migration
operations. Moreover, popularity-based balancing strategies often
make aggressive migration decisions [39], which can hinder the
full utilization of cluster resource. Ultimately, Origami is able to
accurately identify subtrees with higher migration benefits and
strikes a better trade-off between metadata load and locality preser-
vation. As a result, it increases metadata throughput to 75.0k with
five MDSs, which is 3.86×that of a single MDS and 1.73×as high
as the best-performing baseline, C-Hash.
Average latency under single thread.We then re-ran Trace-A
using a single thread to quantify the degree of disruption to names-
pace locality under different strategies. As shown in Figure 5b,
single MDS, which does not involve load balancing, achieved the
lowest latency since all operations could be completed with a single
RPC without additional overhead. In contrast, C-Hash and F-Hash
have increased the latency of metadata operations by 43.9% and
89.1%, respectively. This is because as the number of hash opera-
tions increases, the average number of forwarding steps required
for each metadata operation also increases, which degrades the
performance of metadata under low-load conditions. In contrast to
hashing methods, ML-tree and Origami do not migrate metadata
too aggressively, resulting in latency increases of 29.3% and 24.2%
compared to a single MDS.
The overall performance experiments validate that Origami out-
performs other methods by precisely predicting migration benefits,

ICPP ’25, September 08–11, 2025, San Diego, CA, USAYiduo Wang, Wenda Tang, Linghang Meng, Liang Li, and Jie Wu
0.00
0.25
0.50
0.75
1.00
QPSRPCInodesBusyTime
Single MDS
Imbalance Factor
C-HashF-HashML-treeOrigami
0.00
0.25
0.50
0.75
1.00
QPSRPCInodesBusyTime
Single MDS
Imbalance Factor
Figure 6: The imbalance factors of different balancing strate-
gies on 4 metrics (lower means better balance).
achieving a better trade-off between metadata load balancing and pre-
serving namespace locality.Origami maximizes metadata through-
put during high loads while minimizing performance degradation
during low loads.
5.3  Balance Analysis
Furthermore, we evaluate load balancing using theImbalance Fac-
tor[39], which ranges from 0 to 1, with higher values indicating
greater imbalance. For example, in a cluster with 5-MDSs, an Imbal-
ance Factor of 1 means that all requests go to a single MDS. To dive
into the namespace partition, we further extend the imbalance fac-
tor fromQPSto other metrics:RPCs(number of RPCs handled per
epoch),Inodes(number of metadata entries stored), andBusyTime
(total metadata processing time per epoch).
We first analyze the balance of metadata requests using the im-
balance factor. As Figure 6 shows, we found that even the most
imbalanced C-Hash managed to keep the imbalance factor at a
relatively low value. Although F-Hash achieved the best balancing
effect, the imbalance factor only decreased from 0.37 to 0.33. The
values of the imbalance factor for ML-tree and Origami were inter-
mediate. Furthermore, we dive into the distribution of metadata RPC
and Inodes. Similarly to QPS, F-Hash attained the lowest imbalance
factor values in both of these metrics, indicating that the hashing
method can effectively distribute the metadata evenly, albeit at a
considerable performance downgrade. These results validate our
conclusion: evenly partitioning metadata is not the optimal strat-
egy, and we should trade-off should be made between namespace
locality and load balancing carefully.
To fully understand the load balancing of the MDS cluster, we
measured the cumulative time that each MDS was spent process-
ing client metadata requests over every epoch. For the hash-based
methods, the balance of BusyTime was similar to previous findings.
Interestingly, we found that ML-tree, due to its less aggressive parti-
tioning of metadata, resulted in some MDSs having very low loads,
which led to the highest imbalance factor. Surprisingly, Origami
exhibited a very low imbalance factor in BusyTime, reducing it by
48.3% compared to F-hash, indicating that all its MDSs utilized the
resource to process metadata at a relatively high level. This obser-
vation highlights one reason for Origami’s superior performance:
Ensuring all MDSs busy is more efficient than evenly partitioning.
Table 2: Aggregated metadata throughput and per-request
RPC count: comparison with and without metadata cache.
Throughput# RPC per request
w/o cachew/ cachew/o cachew/ cache
C-Hash32.8±3.3 k46.0±3.0 k2.23±0.031.54±0.02
F-Hash22.5±2.0 k30.0±1.3 k2.87±0.112.27±0.07
ML-Tree26.7±3.7 k38.6±2.3 k1.62±0.021.17±0.02
Origami39.3±3.7 k78.9±7.8 k1.85±0.021.04±0.01
5.4  Metadata Cache Analysis
In this section, we assess the effects of caching on various systems
by toggling the near-root cache on and off, and we analyze how
their choices differ in the selection of subtrees for migration.
Improving aggregated throughput.Table 2 shows that near-root
caching significantly improves performance across all baselines by
reducing path resolution overhead with minimal synchronization,
thus easing MDS pressure. Without caching, C-Hash, F-Hash, and
ML-tree only improve single MDS throughput by 68.8%, 19.9%, and
42.2%, far below the ideal 5×scaling. In contrast, Origami achieves
2.09×throughput without caching, demonstrating more efficient
hardware utilization despite root node hotspots. With near-root
caching enabled, the throughput of C-Hash, F-Hash, and ML-tree in-
creased by 40. 5%, 33. 3%, and 44. 7%, respectively, indicating limited
scalability. In contrast, Origami experienced a 100.7% throughput
increase, as its performance is mainly limited by near-root node
overload rather than path resolution or partition imbalance.
Reduce path resolution overhead.We further measured average
RPCs per metadata operation with and without caching. Without
caching, C-Hash and F-Hash see a significant increase in average
RPCs per request, reaching 2.23 and 2.87, respectively. In contrast,
the ML-tree approach is more conservative, increasing the RPC
forwarding count by only 0.617×, albeit at the cost of less optimal
load balancing. Origami strikes a balance between these extremes,
increasing the RPC count by 0.85×while still delivering overall
performance improvements. After enabling the cache, RPCs per
metadata request decreased by 0.68 to 1.17 for the baseline, but
high throughput and low forwarding overhead cannot be achieved
simultaneously. Surprisingly, Origami ’s extra RPC per request fell
to just 0.035 after caching, outperforming all baselines.
To understand this advantage, we analyzed Origami’s migra-
tion decisions and found that it has a particular inclination toward
migrating two types of subtree: (1)subtrees that are near the root
node and have a high load, which can significantly improve the
balance of the cluster with just a single migration; (2)subtrees that
are far from the root node and write-intensive, which migration only
impacts a small amount of metadata operations, but yield substan-
tial balancing benefits. Therefore, the near-root cache significantly
benefits Origami, as most migrations occur in cached areas. Each
metadata operation adds only 0.03 additional RPCs for path resolu-
tions, making the overhead from migrations negligible compared
to the benefits.

Origami: Efficient ML-Driven Metadata Load Balancing for Distributed File SystemsICPP ’25, September 08–11, 2025, San Diego, CA, USA
0%
25%
50%
75%
100%
051015
Normalized E
ffi
ciency
051015
051015
051015
Time(min)Time(min)Time(min)Time(min)
M1M2M3M4M5
(a) C-Hash
M1
M2
M3
M4
M5
0%
25%
50%
75%
100%
051015
Normalized E
ffi
ciency
051015
051015
051015
Time(min)Time(min)Time(min)
Time(min)
(b) F-Hash
M1
M2
M3
M4
M5
0%
25%
50%
75%
100%
051015
Normalized E
ffi
ciency
051015
051015
051015
Time(min)Time(min)Time(min)
Time(min)
(c) ML-tree
M1
M2
M3
M4
M5
0%
25%
50%
75%
100%
051015
Normalized E
ffi
ciency
051015
051015
051015
Time(min)Time(min)Time(min)
Time(min)
(d) Origami
Figure 7: Efficiency comparison, where efficiency refers to the proportion of time each MDS
spends processing metadata, normalized to a single MDS setup.
1
2
3
4
12345
Normalized Throughput
MDS
Linear
C-Hash
F-Hash
ML-tree
Origami
Figure 8: Scalability Com-
parison.
5.5  Efficiency and Scalability
Higher Efficiency.In Figure 7, we show the efficiency for the
first 15 minutes of each strategy. Although hash-based techniques
enable parallel processing of metadata from the beginning, their
efficiency is considerably worse compared to a single MDS setup.
This is due to the high volume of forwarding requests that must be
handled and the difficulty in achieving ideal balancing. The other
two systems gradually migrate subtrees between MDSs. However,
ML-tree faces significant extra overhead to achieve load balancing.
In contrast, Origami efficiently and progressively transfers metadata
with minimal degradation in efficiency.
Better Scalability.We compare the scalability by measuring the
aggregated throughput as the number of MDSs increases from 2
to 5, with all results normalized to the performance of a single
MDS. Since balance and efficiency are difficult to trade off, none of
the baseline strategies scales effectively. For F-hash with 4 MDS,
although hashing improves balance, this benefit is offset by the
overhead from reduced locality. However, Origami demonstrates a
distinct performance characteristic, as aggregate throughput with
three MDSs reaches 2.7 times that of a single MDS, showing nearly
linear scalability. As more MDSs are added, this trend slows slightly
due to the increased overhead associated with finer-grained load
balancing. In general, Origami achieves near-linear scalability.
5.6  Real-world Workload Results
We replayed traces from 3 real-world workloads with distinct char-
acteristics: Read-Write, Read-Only, and Write-Intensive. We first
measured the throughput focus soley on metadata, then enabled
the data path and evaluated the end-to-end filesystem throughput.
First, Figure 9a presents a comparison of metadata throughput.
Compared to baseline, Origami consistently achieves the highest
throughput, with improvements ranging from 12.5% to 102.9%. Al-
though the applicability of different baseline strategies varies be-
tween traces, Origami shows improvements of 73. 3%, 54. 3%, and 12.
5% over the second-best baseline, respectively. Origami performs
worst on Trace-WI due to the highly dynamic and skewed load,
which complicates balancing; however, it still shows a significant im-
provement over the baseline strategy. Next, we present the end-to-
end throughput after enabling the data path, as shown in Figure 9b.
Origami still delivered the best performance, increasing the meta-
data throughput of the second-best baseline from 1.11×to 1.37×.
The absolute value of end-to-end data throughput is somewhat
0.00
0.25
0.50
0.75
1.00
QPSRPCInodesBusyTime
Single MDS
Imbalance Factor
C-HashF-HashML-treeOrigami
0
20K
40K
60K
80K
RWROWI
Aggregated QPS
(a) Metadata Only
0
20K
40K
60K
80K
RWROWI
Aggregated QPS
(b) End-to-End
Figure 9: The aggregated throughput for three real-world
traces, both w/o and w/ the data path.
lower compared to metadata throughput, as expected. Moreover,
if we allocate additional hardware resources to data service–as is
common in production systems–it can be anticipated that Origami
could further increase the end-to-end performance.
6  Related Work
Metadata partitioning and load balancing.Modern DFSs gen-
erally separate metadata from data and distribute metadata across
multiple MDSs to achieve scalability. Lustre, InfiniFS, Tectnoic [25,
26,30] hashes metadata based on identifiers like f

[... truncated at 50,000 chars. Total available: 63,091 chars]