一、根据分子标记建ML树的流程
1.确定外群（outgroup）
2.确定分子标记(此例为cox1)
3.根据物种accession列，生成t.info文件（accession number和物种名用Tab键格开）
4.根据t.info文件下载（包括outgroup物种）和目标物种cds多序列文件

软件及安装：
1.asn2all:下载核苷酸，氨基酸，genbank文件
sudo apt install ncbi-tools-bin
2.python:pythonIO
sudo apt install python
3.R:分析与作图
sudo apt install r-base
4.muscle:序列比对
java环境，不要安装最新的10版本，很多软件无法支持
apt install muscle
5.samtools:sqn文件转核苷酸序列，氨基酸序列，生成bam.bai文件
apt install samtools
6.blastn:本地序列比对
7.fastqc
8.seqtk:取子序列，如从fasta文件中取出对应id的序列，或者前10条序列
9.awk:文件行处理
10.sed：shell文件处理


mkdir cds
awk ‘{print $1}’t.info | while read a; do asn2all -r -A $a -o cds/$a.cds -f d; done

5.将目标物种cds多序列文件放到cds目录下，这时所有物种的序列都在一起进行分析
6.这时会将cds文件夹下各文件中cox1提取出来，在当前目录下生成一个cox1.fsa的文件
python ~/luria/getcox1_V2.py cds
#cds是cds文件存放的文件夹

7.多序列比对：如果选择用RAxML来建ML树，则需用MEGA进行比对后再转化为phylip格式（*.phy），RAxML只能识别这种比对格式文件，MEGA比对掐头去尾后输出FASTA格式（*.fas）文件（同时也要生成*.nex格式文件用于后期检测饱和度）
8.将fasta文件转为phy文件格式
python fa2plyp.py dna.fa dna.phy
fa2plyp.py文件：
import os
import sys
import re
(xx,a,b)=sys.argv
with open(a ,'r') as fin:
    sequences = [(m.group(1), ''.join(m.group(2).split()))
    for m in re.finditer(r'(?m)^>([^ \n]+)[^\n]*([^>]*)', fin.read())]
with open(b, 'w') as fout:
    fout.write('%d %d\n' % (len(sequences), len(sequences[0][1])))
    for item in sequences:
        fout.write('%-14s %s\n' % item)

9.替代饱和检测
10.最佳替代模型的选择
11.建ML树，可选软件有RAxML，Treefinder, Phyml(据说是最快的进化ML树的软件)，Garli。综合软件有PHYLIP，MEGA，PAUP。（虽然MEGA也可以做ML，但模型太少，不建议。）在集群上我们使用的RAxML。
12.核苷酸进化树如下:
vim建一个名为ml.dna.sh的脚本，脚本里写
ml.dna.sh文件：
#$ -S /bin/sh
/evolution/raxml -s nucl.phy -m GTRGAMMAI -f a -N 1000 -T 8 -x 855500 -p 23444 -n dna_b1000
#evolution/raxml是raxml软件的安装目录的命令，nucl.phy是核苷酸转成的phylip文件，GTRGAMMAI表示最优模型为GTR+GAMMA+I,-m 后接算出的最佳模型-t 后接cpu数，保存后，用qsub -cwd -l vf=10G ml.dna.sh抛到集群去运算如果有报错，错误原因会写在“*.o*”的文件中。


保存等值长为Topologic
非等值长为Phylogentic

如是氨基酸树则如下：
#$ -S /bin/sh
/evolution/raxml -s aa.phy -m PROTGAMMAILGF -f a -N 1000 -T 15 -x 855500 -p 23444 -n dna_b1000
#evolution/raxml是raxml软件的安装目录的命令，nucl.phy是氨基酸序列转成的phylip文件，PROTGAMMAILGF表示最优模型为MtMam+I+G+F,-m 后接算出的最佳模型-t 后接cpu数，保存后，用qsub -cwd -l vf=10G ml.dna.sh抛到集群去运算如果有报错，错误原因会写在“*.o*”的文件中。

13.将accession号替换为物种名
vim建一个名为ch.sh的脚本，脚本里写
#$ -S /bin/sh
cp RAxML_bipartitionsBranchLabels.dna_b1000  tree.nwk
awk '{print $1,$2,$3}' t.info | while read a b c ;do echo "sed -i \"s/${a}/'${b} $c ($a)'/\" tree.nwk ";done > s.sh
#Taxonomy.info.xls是t.info时要注意物种名是否有更改，如是tox.xls则要注意是否需要NC号后面的.1，执行完生成一个原始的tree.nwk和s.sh文件

14.把tree.nwk文件导入mega中进行树的可视化修改
有关Bootstrap和Jackknifing
bootstrap:以每一列为单位，随机排放各列(行的长度始终相等)，产生不同的多序列
jackknifing:随机取一半，再随机取一半的一半，再随机取一半的一半的一半...

****************************************************************************
4.安装awk
sudo apt-get install awk

5.将ID按行存入list文件，用asn2all同时获取多个cds核酸序列
awk '{print$1}' list | while read a;do asn2all -r -A $a -o $a.cds -f d;done
(awk根据域块处理文件的语言，print$1（从第一行起逐行读取打印出来），list为awk操作的文件，|（管道，在前步的结果中继续下一步操作），while ...do...循环读取，a为定义的变量相当于逐行改变的ID，调用变量a用$a,-A的参数为$a,-o的参数为$a.cds方便的核酸序列命名，-f的参数为d,用分号;结束语句，done结束循环）

6.将ID按行存入list文件，用asn2all同时获取多个氨基酸序列
awk '{print$1}' list | while read a;do asn2all -r -A $a -v $a.faa -f d;done
（-v的参数为$a.faa方便的氨基酸序列命名）

7.将ID按行存入list文件，用asn2all同时获取多个genbank文件
awk '{print$1}' list | while read a;do asn2all -r -A $a -o $a.gbk -f d;done
（-o的参数为$a.gbk是genbank文件命名）

8.使用cat命令从文件中读入两个文件，然后将重定向到一个新的文件。
用法示例：
将file1.txt和file2.txt合并到file.txt
$ cat file1.txt file2.txt > file.txt

也可以只使用cat命令读入一个文件，然后使用>>将文本流追加到另一个文件的末位。
用法示例：
将file1.txt追加到file2.txt的末尾
$ cat file1.txt >> file2.txt

9.将同一目录下相同格式(.txt .faa .cds)的文件以换行合并在新的文件中
cat *.txt > new.txt

10.修改系统时间
sudo date 042615202018.00
（必须用root权限，date命令，参数MMDDhhmmYYYY.ss）

11.安装muscle用来多序列比对
sudo apt-get install muscle

12.将seqdump.txt中多个序列比对，输出fast格式，便于构树
muscle -in seqdump.txt -out seq.fa
（-in 指输入，-out输出，对象为对应的文件，输出文件可任意命名）

13.多序列比对，输出为clusterw格式，便于查看
muscle -in seqdump.txt -out seq.txt -clw
（-in 指输入，-out输出，对象为对应的文件，输出文件可任意命名）

14.安装Java环境，并配置环境变量，尽量配置在/etc/profile中的系统变量/usr/benben/.bashrc为当前用户变量
验证Java安装完成：1)java命令 2）javac命令

15.将fasta格式转为phylip格式
1）先安装python2.x版本环境
2）安装biopython包，验证包安装注意：import Bio(不是import biopython)
3）python ./fa2phy.py name.fa name.phy
(即将fa文件转为phy文件，注意fa2phy.py的相对路径,现在为当前文件夹）

16.下载Jmodletest.jar文件，用于构建模型)
1）java -jar jModelTest.jar -help
(查看帮助，是-help 不是--help)

2）Example: java -jar jModeltest.jar -d name.phy -i -f -g 4 -BIC -AIC -AICc -DT -v -a -w
(这的name.phy需要替换为自己的phylip文件名，注意jModeltest.jar的相对位置）

3）java -jar ./jModelTest.jar -d name.phy -i -f -g 4 -BIC -AIC -AICc -DT -v -a -w -BIC
（这个可以执行完成，上个命令可能cannot access）

java -jar ./jModelTest.jar -d seq.phy -i -f -g 4 -BIC -AIC -AICc -DT -v -a -w -BIC
运行结果出来的模型

在AKAIKE INFORMATION CRITERION (AIC) 处AIC表示最佳，AICc，BIC次之
AIC处核酸模型为GTR+G最优模型 GTR+G=GTRGAMMA 用于-m的模型参数

Model selected: 
   Model = GTR+G
   partition = 012345
   -lnL = 3231.4514
   K = 37
   freqA = 0.3138 
   freqC = 0.1785 
   freqG = 0.2425 
   freqT = 0.2652 
   R(a) [AC] =  6.4085
   R(b) [AG] =  31.4451
   R(c) [AT] =  4.1943
   R(d) [CG] =  1.1661
   R(e) [CT] =  50.1875
   R(f) [GT] =  1.0000
   gamma shape = 0.7140 

17.模型认识核酸和氨基酸（蛋白质）模型不同

核酸的部分模型

AIC MODEL SELECTION : Selection uncertainty
 
Model             -lnL    K      AIC       delta       weight   cumWeight
------------------------------------------------------------------------- 
GTR+G       3231.45141   37  6536.902820   0.000000    0.424639   0.424639 
GTR+I       3231.51238   37  6537.024760   0.121940    0.399522   0.824160 
GTR+I+G     3231.44304   38  6538.886080   1.983260    0.157529   0.981689 
GTR         3235.66839   36  6543.336780   6.433960    0.017018   0.998707 
HKY+G       3242.08422   33  6550.168440  13.265620    0.000559   0.999266 
HKY+I       3242.17284   33  6550.345680  13.442860    0.000512   0.999778 
HKY+I+G     3242.07736   34  6552.154720  15.251900    0.000207   0.999985 
HKY         3246.67944   32  6557.358880  20.456060   1.53e-005   1.000000 
SYM+G       3262.21706   34  6592.434120  55.531300   3.71e-013   1.000000 
SYM+I       3262.31269   34  6592.625380  55.722560   3.37e-013   1.000000 
SYM+I+G     3262.21393   35  6594.427860  57.525040   1.37e-013   1.000000 
SYM         3266.36836   33  6598.736720  61.833900   1.59e-014   1.000000 
K80+G       3270.81576   30  6601.631520  64.728700   3.74e-015   1.000000 
K80+I       3270.91518   30  6601.830360  64.927540   3.38e-015   1.000000 
K80+I+G     3270.81340   31  6603.626800  66.723980   1.38e-015   1.000000 
K80         3274.94698   29  6607.893960  70.991140   1.63e-016   1.000000 
F81+I       3367.07359   32  6798.147180  261.244360   7.93e-058   1.000000 
F81+G       3367.09718   32  6798.194360  261.291540   7.75e-058   1.000000 
F81+I+G     3367.08493   33  6800.169860  263.267040   2.89e-058   1.000000 
F81         3369.75482   31  6801.509640  264.606820   1.48e-058   1.000000 
JC+I        3394.89697   29  6847.793940  310.891120   1.31e-068   1.000000 
JC+G        3394.92949   29  6847.858980  310.956160   1.27e-068   1.000000 
JC+I+G      3394.91775   30  6849.835500  312.932680   4.74e-069   1.000000 
JC          3397.64046   28  6851.280920  314.378100   2.30e-069   1.000000



18.根据模型做进化树RAXML详解
在RAxML官网下载Linux版本的软件

PDF帮助文件有详细的解压，编译和使用方法
gunzip standard­RAxML­8.0.0.tar.gz 
tar xf standard­RAxML­8.0.0.tar

这里使用第一种编译方法，编译两次后最后产生raxmlHPC-PTHREADS-AVX文件
cd standard­RAxML­8.0.0/
ls
ls Makefile.*
make ­f Makefile.AVX.gcc 
rm *.o 
make ­f Makefile.AVX.PTHREADS.gcc 
产生raxmlHPC-PTHREADS-AVX即是命令需要使用的文件

所以运行：raxmlHPC-PTHREADS-AVX -x 12345 -p 12345 -# 100 -m GTRGAMMA -T 4 -s align_file.phy -n TEST(未出树）
        raxmlHPC-PTHREADS-AVX -f a -x 12345 -p 12345 -# 100 -m GTRGAMMA -T 4 -s seq.phy -n TEST（完成）
（注意这里的调用文件的相对路径和绝对路径）

下载和安装
RAxML 可以在 Linux, MacOS, DOS 下运行，下载网址为自己百度
也可以使用phylobench.vital-it.ch/raxml-bb/ 在线运行。 对于 Linux 和 Mac 用户下载 RAxML-7.0.4.tar.gz 用 gcc 编译即可, make –f Makefile.gcc。Windows 用户可以下载编译好的 exe 文件，而无需安装。
2 数据的输入
RAxML 的数据为 PHYLIP 格式，但是其名字可以增加至 256 个字符。“RAxML 对
PHYLIP 文件中的 tabs，inset 不敏感”。输入的树的格式为 Newick
RAxML 的查错功能
序列的名称有重复，即不同的碱基却拥有一致的名称。
序列的内容重复，即两条不同名称的序列，碱基完全一致。
某个位点完全由序列完全由未知符号组成，如氨基酸序列完全由 X,?,*,-组成，DNA 序列完全由 N,O,X,?,-组成。
序列完全由未知符号组成，如氨基酸序列完全由 X,?,*,-组成，DNA 序列完全由 N,O,X,?,-组成。
序列名称中禁用的字符 如包括空格、制表符、换行符、:,(),[]等
3 RAxMLHPC 下的选项参数以及用法
-s sequenceFileName 要处理的 phy 文件
-n outputFileName 输出的文件
-m substitutionModel 模型设定 方括号中的为可选项：
[-a weightFileName] 设定每个位点的权重，必须在同一文件夹中给出相应位点的权重
[-b bootstrapRandomNumberSeed] 设定 bootstrap 起始随机数
[-c numberOfCategories] 设定位点变化率的等级
[-d] -d 完全随机的搜索进化树，而不是从 maximum parsimony tree 开始。在 100 至 200 个分类单元间，该选可能会生成拓扑结构完全不同的局部最大似然树。
[-e likelihoodEpsilon]默认值为 0.1
[-E excludeFileName] 排除的位点文件名
[-f a|b|c|d|e|g|h|i|j|m|n|o|p|s|t|w|x] 算法
-f a rapid Bootstrap
-f b draw the bipartitions using a bunch of topologies
-f c checks if RAxML can read the alignment.
-f d rapid hill-climbing algorithm
-f e optimize the model parameters
-f g compute the per–site log Likelihoods for one ore more trees passed via -z.
-f h compute a log likelihood test (SH-test [21]) between a best tree passed via -t and a bunch of other trees passed via -z.
-f i performs a really thorough standard bootstrap
[-g groupingFileName] 预先分组的名称
[-h] program options
[-i initialRearrangementSetting] speccify an innitial rearrangement setting for the ininital phase of the search algorithm.
[-j]
[-k] optimize branchlength and model parameters on bootstrapped trees
[-l sequenceSimilarityThreshold] Specify a threshold for sequence similarity clustering. [-L sequenceSimilarityThreshold]
[-M] 模型设定
-m GTRCAT: GTR approximation
-m GTRMIX: Search a good topology under GTRCAT
-m GTRGAMMA: General Time Reversible model of nucleotide subistution with the gamma model of rate heterogeneity.
-m GTRCAT_GAMMA: Inference of the tree with site-specific evolutionary rates. 4 discrete GAMMA rates,
-m GTRGAMMAI: Same as GTRGAMMA, but with estimate of proportion of invariable sites
-m GTRMIXI: Same as GTRMIX, but with estimate of proportion of invariable sites.
-m GTRCAT GAMMAI: Same as GTRCAT_GAMMA, but with estimate of proportion of invariable sites.
-n outputFileName 输出文件名
-o outgroupName(s) 设定外类群 如果有两个以上外类群，两者之间不能用空格，而应该用英文的"," DNA, gen1=1-500 DNA, gen2=501-1000
[-p parsimonyRandomSeed] [-P proteinModel]
[-q multipleModelFileName]
-q multiple modelfile name
如将以下信息拷贝到另存为文件 genenames
DNA, rbcLa = 1-526
DNA, matK = 527-1472
调用方法 -q genenames
-m GTRGAMMA
[-r binaryConstraintTree]
-s sequenceFileName 待分析的 phy 文件
[-t userStartingTree] 用户指定的进化树拓扑结构
[-T numberOfThreads]
[-u multiBootstrapSearches] Specify the number of multiple BS searches per replicate to obtain betterML trees for each replicate. [-v] 版本信息
[-w workingDirectory] 将文件写入的工作目录
[-x rapidBootstrapRandomNumberSeed] invoke rapidBootstrap
[-y] -y 只输出简约树拓扑结构，之后推出，该树也可以用于 GARLI 等软件
[-z multipleTreesFile]
[-#|-N numberOfRuns]
生成的文件
RAxML log.exampleRun: 运行时间、似然值/ number of checkpoint file
RAxML result.exampleRun: 树文件
RAxML info.exampleRun: -m GTRGAMMA or -m GTRMIX contains information about the model and algorithm used
RAxML parsimonyTree.exampleRun: -t.
RAxML randomTree.exampleRun: -d.
RAxML checkpoint.exampleRun.checkpointNumber: -j
RAxML bootstrap.exampleRun: -# and -b or -x
RAxML bipartitions.exampleRun: -f b
RAxML reducedList.exampleRun: -l or -L
RAxML bipartitionFrequencies.exampleRun: -t , -z , -f m
RAxML perSiteLLs.exampleRun: -f g
RAxML bestTree.exampleRun: -x 12345 -f a
RAxML distances.exampleRun: -f x 


19.构建进化树
1. PhyML
构建进化树方法：Maximum Likelihood
评估：选择bootstrap或者Likelihood-ratio test
运行方式：所有平台和网页
心得：理论上支持4000条序列，小于2000000个字符。但是，对于个人电脑，通常100-200条序列比较合适。命令行运行时，可以选择非常简介的默认模式运行。在默认模式下，bootstrap需要手动开启。安装和使用非常方便，直接下载后可以直接使用。同时，bootstrap可以通过MPI分布计算，但是需要从源代码安装。
快速运行：phyml -i align_file.phy --no_memory_check
-i：后跟需要Phylip格式文件
--no_memory_check：不用检查内存，防止程序运行时跳出

2. RAxML
构建进化树方法：Maximum Likelihood
运行方式：所有平台和网页。
心得：网页运行推荐 http://www.phylo.org/portal2/login!input.action ，支持数据的存放和其他构建进化树的方法。本地安装支持MPI和PThreads的分布计算，但是安装有些复杂，需要仔细阅读文档。
快速运行1：raxmlHPC-PTHREADS-AVX -x 12345 -p 12345 -# 100 -m GTRGAMMA -T 4 -s align_file.phy -n TEST
-x：bootstrap运行时设定随机数，用于结果重现
-p：parsimony推断时设定随机数，用于结果重现
-#：bootstrap次数。也可以设定为autoMRE，最大次数是1000。
-m：设定使用的模型，GTRGAMMA为核苷酸序列适用模型
-T：设定线程数，不要超过最大线程
-s：输入文件，Phylip或者fasta文件
-n：输入文件记号
快速运行2：raxmlHPC-PTHREADS-AVX -f a -x 12345 -p 12345 -# autoMRE -m GTRCAT -T 4 -s align_file.phy -n TEST
-f a：选定算法，快速bootstrap

20.最后产生的文件5个
RAxML_bestTree.TEST
RAxML_bipartitions.TEST
RAxML_bipartitionsBranchLabels.TEST
RAxML_bootstrap.TEST
RAxML_info.TEST

在mega中查看RAxML_bestTree.TEST的进化树结构
==========================================================================
LINUX命令

21.top 查看进程和cup占用
22.df查看磁盘信息，df --help 
-a, --all             include pseudo, duplicate, inaccessible file systems
  -B, --block-size=SIZE  scale sizes by SIZE before printing them; e.g.,
                           '-BM' prints sizes in units of 1,048,576 bytes;
                           see SIZE format below
  -h, --human-readable  print sizes in powers of 1024 (e.g., 1023M)
  -H, --si              print sizes in powers of 1000 (e.g., 1.1G)
  -i, --inodes		显示inode 信息而非块使用量
  -k			即--block-size=1K
  -l, --local		只显示本机的文件系统
      --no-sync		取得使用量数据前不进行同步动作(默认)
      --output[=FIELD_LIST]  use the output format defined by FIELD_LIST,
                               or print all fields if FIELD_LIST is omitted.
  -P, --portability     use the POSIX output format
      --sync            invoke sync before getting usage info
      --total           elide all entries insignificant to available space,
                          and produce a grand total
  -t, --type=TYPE       limit listing to file systems of type TYPE
  -T, --print-type      print file system type
  -x, --exclude-type=TYPE   limit listing to file systems not of type TYPE
  -v                    (ignored)
      --help		显示此帮助信息并退出
      --version		显示版本信息并退出



df -H 以G显示磁盘占用情况

23.建立软链接ln -s path/ori-name path/new-name

24.新建文件
touch fileName
vim nul>fileName

25.grep文本查询
-a 不要忽略二进制数据。
-A<显示列数> 除了显示符合范本样式的那一行之外，并显示该行之后的内容。
-b 在显示符合范本样式的那一行之外，并显示该行之前的内容。
-c 计算符合范本样式的列数。
-C<显示列数>或-<显示列数>  除了显示符合范本样式的那一列之外，并显示该列之前后的内容。
-d<进行动作> 当指定要查找的是目录而非文件时，必须使用这项参数，否则grep命令将回报信息并停止动作。
-e<范本样式> 指定字符串作为查找文件内容的范本样式。
-E 将范本样式为延伸的普通表示法来使用，意味着使用能使用扩展正则表达式。
-f<范本文件> 指定范本文件，其内容有一个或多个范本样式，让grep查找符合范本条件的文件内容，格式为每一列的范本样式。
-F 将范本样式视为固定字符串的列表。
-G 将范本样式视为普通的表示法来使用。
-h 在显示符合范本样式的那一列之前，不标示该列所属的文件名称。
-H 在显示符合范本样式的那一列之前，标示该列的文件名称。
-i 忽略字符大小写的差别。
-l 列出文件内容符合指定的范本样式的文件名称。
-L 列出文件内容不符合指定的范本样式的文件名称。
-n 在显示符合范本样式的那一列之前，标示出该列的编号。
-q 不显示任何信息。
-R/-r 此参数的效果和指定“-d recurse”参数相同。
-s 不显示错误信息。
-v 反转查找。
-w 只显示全字符合的列。
-x 只显示全列符合的列。
-y 此参数效果跟“-i”相同。
-o 只输出文件中匹配到的部分。
文件中搜索一个单词，得到该行
grep "match_pattern" fileName

在多个文件中查找：
grep "match_pattern" file1 file2 file3

输出除之外的所有行 -v 选项：
grep -v "match_pattern" fileName

使用正则表达式 -E 选项：
grep -E "[1-9]+"

只输出文件中匹配到的部分 -o 选项：
grep -o "match_pattern" fileName

统计文件或者文本中包含匹配字符串的行数 -c 选项：
grep -c "text" fileName

grep "text" -n fileName
grep "text" -n file1 file2

搜索多个文件并查找匹配文本在哪些文件中：
grep -l "text" file1 file2

在多级目录中对文本进行递归搜索：
grep "text" ./ -f -n
忽略匹配样式中的字符大小写：
grep -i "text" fileName

选项 -e 制动多个匹配样式：
grep -e "text1" -e "text2" fileName


#只在目录中所有的.php和.html文件中递归搜索字符"main()"
grep "main()" . -r --include *.{php,html}
显示匹配某个结果之后的3行，使用 -A 选项：
seq 10 | grep "5" -A 3
显示匹配某个结果之前的3行，使用 -B 选项：
seq 10 | grep "5" -B 3
显示匹配某个结果的前三行和后三行，使用 -C 选项：
seq 10 | grep "5" -C 3

vim大小写替换
vim中大小写转化的命令是gu或者gU
形象一点的解释就是小u意味着转为小写；大U意味着转为大写.剩下的就是对这两个命令的限定（限定操作的行，字母，单词）等等
~切换光标所在位置的字符的大小写形式，大写转换为小写，小写转换为大写
3~将光标位置开始的3个字母改变其大小写
gu：切换为小写，gU：切换为大写，剩下的就是对这两个命令的限定（限定行字母和单词）

    ggguG：整篇文章转换为小写，gg：文件头，G：文件尾，gu：切换为小写
    gggUG：整篇文章切换为大写，gg：文件头，G：文件尾，gU：切换为大写
只转化某个单词
    guw、gue
    gUw、gUe
    gu5w：转换 5 个单词
    gU5w
转换行
    gU0 ：从光标所在位置到行首，都变为大写
    gU$ ：从光标所在位置到行尾，都变为大写
    gUG ：从光标所在位置到文章最后一个字符，都变为大写
    gU1G ：从光标所在位置到文章第一个字符，都变为大写

************************************************************************
dpkg: 处理软件包 xxx (--configure)时出错 解决办法
第一步：备份
$ sudo mv /var/lib/dpkg/info /var/lib/dpkg/info.bk
第二步：新建
$ sudo mkdir /var/lib/dpkg/info
第三步：更新
$ sudo apt-get update
$ sudo apt-get -f install
如果依旧报错，是由于没有info文件夹的情况，运行提示的那个命令，sudo dpkg --configure -a,这时会报错没有info的文件夹，同时注意apt和apt-get的命令有点不同
第四步：替换
$ sudo mv /var/lib/dpkg/info/* /var/lib/dpkg/info.bk
//把更新的文件替换到备份文件夹
第五步：删除
$ sudo rm -rf /var/lib/dpkg/info
//把自己新建的info文件夹删掉
第六步：还原
$ sudo mv /var/lib/dpkg/info.bk /var/lib/dpkg/info
//把备份的info.bk还原
=========================================================================
二、进化树过程

13种基因类型：
ND1,ND2,ND3,ND4,ND4L,ND4,ND6,COX1,COX2,COX3,ATP6,ATP8,CYTB
muscle是对这13种基因的fasta文件比较的，即一次比较一个文件种多个物种的同一种类型的基因，也可以单独只比对一个基因的文件

1.step1.sh文件：核苷酸，氨基酸，genbank下载
#! -S /bin/sh
#新建对应文件夹
mkdir -p tox/gb nucl/nucl aa/aa

cd tox/gb
#根据accession列下载genbank文件,id.list和t.info不同之处在于t.info有第二列的物种名而原始的list只有accession且不包括目标种
awk '{print $1}' id.list | while read a; do asn2all -r -A $a -f g > $a.gb; done
#将目标种的.sqn文件转为genbank文件格式，这样就避开用目标种的accession下载序列
asn2all -i ../../*.sqn -f g > submitID.gb
#getTox是根据genbank文件生成tox.xls汇总表，前两列是accession和物种名
python ../../Phylotools/getTox_2.py *
mv tox.xls ../
cd ../../

cd nucl/
#下载核苷酸序列文件
awk '{print $1}' ../t.info | while read a; do asn2all -r -A $a -o nucl/$a.nucl -f d; done
#将目标种的.sqn文件转为核苷酸文件格式
asn2all -i *.sqn -o nucl/nucl/submitID.nucl -f d
cd ..

cd aa/
#下载氨基酸序列文件
awk '{print $1}' ../t.info | while read a; do asn2all -r -A $a -v aa/$a.aa -f d; done
#将目标种的.sqn文件转为氨基酸文件格式
asn2all -i *.sqn -v aa/aa/submitID.aa -f d
cd ..

******
2.step2.sh文件,muscle比对
#! /bin/sh
cd nucl/nucl
#取出不同物种相同的基因序列放在同一文件中，1和2是选项参数，一般的物种有13种基因类型，所以产生13个文件
13种基因（ND1,,,
python ../../Phylotools/preTree4nucl.py ../../t.info 1
#muscle比对
ls *.fa | while read l;do muscle -in $l -out $l.ms;done
#生成用于建进化树的核苷酸fsa文件
python ../../Phylotools/preTree4nucl.py ../../t.info 2 > nucl.fsa
cd ../../

cd aa/aa
#取出不同物种相同的氨基酸序列放在同一文件中，1和2是选项参数
python ../../Phylotools/preTree4aa.py ../../t.info 1
#muscle比对
ls *.fa | while read l;do muscle -in $l -out $l.ms;done
#生成用于建进化树的氨基酸fsa文件
python ../../Phylotools/preTree4aa.py ../../t.info 2 > aa.fsa
cd ../../

#创建用核苷酸和氨基酸建进化树的文件夹
mkdir aa/gblocks
mkdir nucl/gblocks
cp aa/aa/aa.fsa aa/gblocks
cp nucl/nucl/nucl.fsa nucl/gblocks

cd nucl/gblocks
#将fasta格式的文件转化为phylip格式的文件，用于建立进化树
python ../../evolution/fa2plyp.py nucl.fsa nucl.phy
cp ../../Phylotools/modeltest.sh .
#在大型机上使用modeltest对核苷酸建立模型之后选择最优模型
qsub -cwd -l vf=55g modeltest.sh
cd ../../

cd aa/gblocks
#将fasta格式的文件转化为phylip格式的文件，用于建立进化树
python ../../evolution/fa2plyp.py aa.fsa aa.phy
cp ../../Phylotools/prottest.sh .
#在大型机上使用prottest对氨基酸建立模型之后选择最优模型
qsub -cwd -l vf=55g prottest.sh
cd ../../

step3.sh文件，
#! -S /bin/sh
cd aa/gblocks
cp RAxML_bipartitionsBranchLabels.dna_b1000 ML_aa.nwk
#将accession号改为对应的物种名
awk -F '\t' '{print $1,$2}' ../../t.info | while read a b; do echo "sed -i \"s/$a/'$b ($a)'/g\" ML_aa.nwk"; done > s.sh
sh s.sh
#sed -i为写入，'s///g'为行替换
sed -i 's/ (submitID)//g' ML_aa.nwk
cd ../../
mkdir aa/ML_aa
cp aa/gblocks/ML_aa.nwk aa/ML_aa

cd nucl/gblocks
cp RAxML_bipartitionsBranchLabels.dna_b1000 ML_nucl.nwk
#将accession号改为对应的物种名
awk -F '\t' '{print $1,$2}' ../../t.info | while read a b; do echo "sed -i \"s/$a/'$b ($a)'/g\" ML_nucl.nwk"; done > s.sh
sh s.sh
#sed -i为写入，'s///g'为行替换
sed -i 's/ (submitID)//g' ML_nucl.nwk
cd ../../
mkdir nucl/ML_nucl
cp nucl/gblocks/ML_nucl.nwk nucl/ML_nucl

=========================================================================

二、根据物种名下载相应的界门纲目科属种信息并提取目科属种作为表格
1.根据物种名列下载所有物种详细信息
sampleInfoName.py文件
#! /usr/bin/env python
import sys
import re
import os
import commands
import time


def main(name_list, output):
    WGET = "/usr/bin/wget"

    OUT = open(output, "w")
    OUT.write("#TAXID\tORGANISM\tORDER\tFAMILY\tGENUS\tALL\n")
    for line in open(name_list):
        line = line.rstrip()
        url_name = "%20".join(line.split())
        print url_name

        cmd_get_taxid = "%s -O rawinfo1.html -T 10 -w 10 http://www.ncbi.nlm.nih.gov/taxonomy/?term=%s" %\
                (WGET, url_name)F
        (status, output) = commands.getstatusoutput(cmd_get_taxid)
        if status != 0:
            sys.stderr.write("ERROR: %d\n%s" % (status, output))
            sys.exit(1)
        print >>sys.stderr, "==================INFO TO DEBUG=============="
        print >>sys.stderr, "SAMPLE: %s\n%s" % (line, output)

        taxid = ""
        for code_line in open("rawinfo1.html"):
            code_line = code_line.rstrip()
            if not re.search("wwwtax\.cgi\?id", code_line):
                continue
            code_infos = code_line.split()
            for i in code_infos:
                if re.search("wwwtax\.cgi\?id", i):
                    taxid = i.split("=")[-1][:-1]
                    print >>sys.stderr, "URL: %s\n" % (i)

        cmd_get_taxonomy = "%s -O rawinfo2.html -T 10 -w 10 http://www.ncbi.nlm.nih.gov/Taxonomy/Browser/wwwtax.cgi?id=%s" %\
                (WGET, taxid)

        time.sleep(5)
        (status, output) = commands.getstatusoutput(cmd_get_taxonomy)
        if status != 0:
            sys.stderr.write("ERROR: %d\n%s" % (status, output))
            sys.exit(1)

        print >>sys.stderr, output

        print >>sys.stderr, "=============================================\n"

        for code_line in open("rawinfo2.html"):
            code_line = code_line.rstrip()
            if re.search("TITLE=\"genus\"", code_line):
                code_infos = code_line.split(";")
                order_val = "unknown"
                family_val = "unknown"
                genus_val = "unknown"
                all_infos = []
                for i in code_infos:
                    if not re.search("TITLE=", i):
                        continue
                    title = i.split("\"")[2]
                    title_val = i.split(">")[1][:-3]
                    all_infos.append(title_val)
                    if title == "order":
                        order_val = title_val
                    if title == "family":
                        family_val = title_val
                    if title == "genus":
                        genus_val = title_val

                OUT.write("%s\t%s\t%s\t%s\t%s\t%s\n" % (taxid, line,\
                        order_val, family_val, genus_val,\
                        "; ".join(all_infos)))

        time.sleep(5)

    OUT.close()

    cmd = "rm -f rawinfo1.html rawinfo1.html"
    (status, output) = commands.getstatusoutput(cmd)
    if status != 0:
        sys.stderr.write("ERROR: %d\n%s" % (status, output))
        sys.exit(1)


if len(sys.argv) > 1:
    main(sys.argv[1], sys.argv[2])
else:
    print >>sys.stderr, "python %s <NameList> <output>" % (sys.argv[0])


2.提取目科属种重要信息

species.py文件
#!/usr/bin/python
import sys
def main(infile,outfile):
    f=open(infile,'r')
    w=open(outfile,'w')

    for i in f:
        i=i.strip().split('\t')
        i2=i[1:6]
        ii="\t".join(i2)
        j=i[5]
        j=j.strip().split(';')
        if j[0]=="<STR":
            for jj in j:
                if jj[-7:-1]=="iforme":
                    i[2]=jj
                elif jj[-4:-1]=="ida":
                    i[3]=jj
                    i[4]=j[-1]
                    i1=i[1:6]
                    iii="\t".join(i1)
                    w.write("%s\n" % iii)
        else:
            w.write("%s\n" % ii)
main(sys.argv[1],sys.argv[2])


=========================================================================

三、不完整的基因来进行构建进化树

1.
ls *.aa |while read a ;do python choice_gene.py gene.list *.aa/*.nucl ;done

2.进行比对
ls *.fa |while read a ;do muscle -in $a -out $a.ms ;done 

3.处理比对后的结果
ls *.ms |while read a ;do python lack.py accession.list $a > $a.out ;done 

ls *.ms |while read a ;do mv $a.out $a ;done

************************************************************************
choice_gene.py文件：
#! /usr/bin/python

import sys
from Bio import SeqIO
clean={
'CO1':'COX1',
'CO2':'COX2',
'CO3':'COX3',
'CYTOCHROME C OXIDASE SUBUNIT 1':'COX1',
'CYTOCHROME C OXIDASE SUBUNIT I':'COX1',
'CYTOCHROME C OXIDASE SUBUNIT 2':'COX2',
'CYTOCHROME C OXIDASE SUBUNIT II':'COX2',
'CYTOCHROME C OXIDASE SUBUNIT 3':'COX3',
'CYTOCHROME C OXIDASE SUBUNIT III':'COX3',
'COX2A':'COX2',
'COI':'COX1',
'COII':'COX2',
'COIII':'COX3',
'COXI':'COX1',
'COXII':'COX2',
'COXIII':'COX3',
'cytb':'CYTB',
'CYTOCHROME B':'CYTB',
'ATPASE 9':'ATP9',
'ATPASE9':'ATP9',
'ATPASE9':'ATP9',
'ATPASE 6':'ATP6',
'ATPASE6':'ATP6',
'ATP SYNTHASE F0 SUBUNIT 6':'ATP6',
'atp9':'ATP9',
'atp6':'ATP6',
'atp8':'ATP8',
'atp 8':'ATP8',
'ATPASE 8':'ATP8',
'ATP 8':'ATP8',
'cytB':'CYTB',
'cob':'CYTB',
'COB':'CYTB',
'CYT B':'CYTB',
'CYT B':'CYTB',
'NADH1':'ND1',
'NADH DEHYDROGENASE SUBUNIT 1':'ND1',
'NADH2':'ND2',
'NADH DEHYDROGENASE SUBUNIT 2':'ND2',
'NADH3':'ND3',
'NADH DEHYDROGENASE SUBUNIT 3':'ND3',
'NADH4':'ND4',
'NADH DEHYDROGENASE SUBUNIT 4':'ND4',
'NADH4L':'ND4L',
'NADH DEHYDROGENASE SUBUNIT 4L':'ND4L',
'NADH5':'ND5',
'NADH DEHYDROGENASE SUBUNIT 5':'ND5',
'NADH6':'ND6',
'NADH DEHYDROGENASE SUBUNIT 6':'ND6',
'NAD1':'ND1',
'NAD2':'ND2',
'NAD3':'ND3',
'NAD4':'ND4',
'NAD4L':'ND4L',
'NAD5':'ND5',
'NAD6':'ND6',
}
def getGene(gene):
	if gene in clean.keys():
		name=clean[gene]
	if gene not in clean.keys():
		name=gene
	return name
def main(gene_file,cds_file):
	name=cds_file.split('.')[0]
	gene_list=[]
	have=[]
	for i in open(gene_file,'r'):
		gene_name=i.strip()
		gene_list.append(gene_name)
	for i in SeqIO.parse(cds_file,'fasta'):
		gene_name=i.description.split('gene=')[-1].split(']')[0].upper()
		new_name=getGene(gene_name)
		if new_name in gene_list and new_name not in have:
			out=open(new_name+'.fa','a')
			have.append(new_name)
			out.write('>%s\n%s\n' % (name,i.seq))
			out.close()
if len(sys.argv) ==3 :
	main(sys.argv[1],sys.argv[2])
if len(sys.argv) !=3:
	print 'python %s <gene.list> <fa>' % sys.argv[0]

************************************************************************
lack.py文件：

#1 /usr/bin/python

import sys
from Bio import SeqIO

def main(accession,fa):
	dic={}
	for i in SeqIO.parse(fa,'fasta'):
		dic[i.id]=i.seq
		length=len(i.seq)
	for i in open(accession,'r'):
		line=i.strip().split()
		name=line[0]
		if name in dic.keys():
			print '>%s\n%s' % (name,dic[name])
		if name not in dic.keys():
			print '>%s\n%s' % (name,'-'*length)
if len(sys.argv) ==3 :
	main(sys.argv[1],sys.argv[2])
if len(sys.argv) !=3:
	print 'python %s <accession.list> <fa>' % sys.argv[0]

========================================================================

四、核外基因组V2流程
核外基因组V2从V1文件夹反馈无误开始，跟据客户特殊需求进行的分析
1.kaks保留4位小数，p-genetic保留3位，0也要加进去，skew保留4位.
2.表格全部标题要加粗，高18，要另存为xls,中文宋体，英文数字time new roman,

1. 进化树
见进化树流程

2. CodonUsage

1.新建一个目录，把/MTG/codonusage的文件夹复制到当前文件夹，cp -r ~/bio/MTG/codonusage ./ 
2.把sqn文件复制到codonusage的文件夹内运行sh step1 *.sqn。程序出来结果的是以第一套密码子做的，所以需要更改密码子。进入CU1,根据物种的所使用的密码子的套数，运行python ../bin/change_codon.py *.csv 2 fre.csv (其中*.csv是本来存在的csv文件，2是密码子的套数，目前程序只支持第2,4,5,22,23这几套的转换，其他的需到程序中添加，fre.csv是出来的结果，名字必须为fre.csv)。
3.用excel打开fre.csv ,先把Frequency一列降序，再Amino acid一列升序（是为了使长的柱子在下面）。在后面新加入一列Countif：=COUNTIF($A$2:A2,A2)
运行sh step2. 生成frequency.pdf和RSCU.pdf。在photoshop中对图片进行调整，另存为pdf,png,tif三种格式。模板如下图：
5.打开RSCU.csv，格式调整为codonusage文件夹下的codonUsage模板.xls的样子，其中字体全文time new roman.1和2列列宽为12,3到5列列宽为16，除了备注行，其余行高为18.
注意：检查时要主要检查stp（即终止密码子），如果stp的数量大于正常的数量时要检查是否有更改密码子的套数

3.tRNA二级结构（可用于发表的）
4. 基因重排
4.1打开网站：http://genome.lbl.gov/vista/mvista/submit.shtml，根据挑选的物种数目+1的数字填入Total number of sequences，然后submit，填入邮箱，inquiry中的sequence1和2导入相同的序列（即目标物种的fasta序列）
4.2 然后把需要做的物种的tbl，然后运行python change_tbl.py *.tbl > *_result.tbl(注意，如果tbl有SSC，LSC或者IR区的注释，要先去掉这几行再运行程序，如需添加这几行，需要自己加进去，程序也没有把orf转变，如果需要添加orf，把程序倒数第三行的‘and’和后面的删掉即可；如有一些中间有其他基因的断裂基因，也需要自己把另外的几段添加进去；添加的规则是：如果是正向则‘>’,反向则’<’,>或<1	100	gene_name，然后下一行，1	100	exon).
4.3在additional options选项中导入上一步生成的tbl文件，注意，物种要对应一开始导入的序列文件的物种。然后点最先面的submit。
4.4等邮件过来，打开链接，下载sequence1的PDF。

********************************************************************

kaks分析(建议客户选用不超过7个物种,如超过7个，最高只做物种）
software: DnaSP
利用DnaSP计算KaKs
序列下载-追加-留id-比对-kaks-fasta_id>name-R图

1.下载核酸序列，一个文件里可能有多个基因
awk '{print $1}' file | while read a; do asn2all -r -A $a -o $a.nucl -f d;done

2.下载genebank文件
awk '{print $1}' file | while read a; do asn2all -r -A $a -f g > $a.gb; done

3.计算一个文件基因数目
grep ">" fileName|wc -l

3.1目标基因除id外除去注释;long_seq1.py
python long_seq1.py file

3.2将目标物种target的6个基因分别对应加进6个文件中，6个基因名和文件名对应
awk '{print $1}' fileList |while read a;do echo ">target" >>$a.fa;grep -A 1 $a 1fastaFile|tail -n 1 >>$a.fa;done

4.取出不同物种相同的基因序列放在同一文件中
python Phylotools/preTree4nucl.py file 1

5.去终止子;kaks_stop.py
ls *.fa|while read a;do python kaks_stop.py $a;done

6.muscle比对;muscle，注意kaks在去终止子后才比对
ls *.fa|while read a;do muscle -in $a -out $a.ms;done

6.1比对完成的序列查看最后---之后是否为3的整数倍，否则应往---前移动

7.在preTree4nucl.py接入第二个参数文件;preTree4nucl.py
python Phylotools/preTree4nucl.py file 2 > nucl.fsa

8.将fasta文件的id号变为物种名;awk
ls *.fa |while read d;do awk '{print $1,$2,$3}' file | while read a b c ;do echo "sed -i \"s/$a/${b}_$c/\" $d ";done > $d.sh;done

ls *.sh|while read a;do sh $a;done

file
NC_028509	Creteuchiloglanis macropterus
NC_028511	Oreoglanis immaculatus
NC_021601	Exostoma labiatum
NC_021600	Glaridoglanis andersonii
NC_021597	Glyptosternon maculatum
NC_021605	Pseudecheneis sulcata
target	Parachiloglanis

9.密码子同义和变义，python调用R 作图;kaks_R.py
ls *.fa|while read a;do python kaks_R.py $a;done


统计
Gene	ID	ka	ks	kaks
ND1	Procypris_rabaudi-Procypris_merus	0.0163826	0.2722762	0.06
ND2	Procypris_rabaudi-Procypris_merus	0.03051462	0.2070977	0.147
ND3	Procypris_rabaudi-Procypris_merus	0.009389947	0.212343	0.044
ND4	Procypris_rabaudi-Procypris_merus	0.02771658	0.3572396	0.078
ND4L	Procypris_rabaudi-Procypris_merus	0.01269962	0.3673707	0.035
ND5	Procypris_rabaudi-Procypris_merus	0.0230239	0.2733556	0.084
ND6	Procypris_rabaudi-Procypris_merus	0.02479636	0.3310648	0.075
ATP6	Procypris_rabaudi-Procypris_merus	0.01261978	0.2788561	0.045
ATP8	Procypris_rabaudi-Procypris_merus	0.009259524	0.2170874	0.043
COX1	Procypris_rabaudi-Procypris_merus	0.003768652	0.2893457	0.013
COX2	Procypris_rabaudi-Procypris_merus	0.008298884	0.2077375	0.04
COX3	Procypris_rabaudi-Procypris_merus	0.01166473	0.2070452	0.056
CYTB	Procypris_rabaudi-Procypris_merus	0.01052227	0.3663134	0.029

*****************************************************************

kaks.py文件
# -*- coding: utf-8 -*-
'''''''''''''''''''''''''''''''''''''''''''''''''''
在获取t.info文件后，生成一系列cds文件，保存到cds文件夹中，
使用本程序可以生成去除终止密码子而且调用了muscle进行了比对
最后每个物种将其全部CDS连成一条
本程序原来用于在计算kaks中整理数据
之所以没有将调用取cds文件的命令行是因为cds文件有可能有错，
特别是目标物种sqn转cds
使用方法：
1. 获取t.info
2. mkdir cds
3. awk -F '\t' '{print $1}' t.info| while read a; do asn2all -r -A $a -o cds/$a.nucl -f d; done
4. asn2all -i *.sqn -o cds/submitID.fa -f d 
5. python kaks.py cds
最后会在工作目录下生成result.fasta
'''''''''''''''''''''''''''''''''''''''''''''''''''
import sys
import re
import os
import argparse
from Bio import SeqIO

__author__ = 'Luria'
__date__= '2016.6.2'

clean_dir={
'CO1':'COX1',
'CO2':'COX2',
'CO3':'COX3',
'CYTOCHROME C OXIDASE SUBUNIT 1':'COX1',
'CYTOCHROME C OXIDASE SUBUNIT I':'COX1',
'CYTOCHROME C OXIDASE SUBUNIT 2':'COX2',
'CYTOCHROME C OXIDASE SUBUNIT II':'COX2',
'CYTOCHROME C OXIDASE SUBUNIT 3':'COX3',
'CYTOCHROME C OXIDASE SUBUNIT III':'COX3',
'COX2A':'COX2',
'COI':'COX1',
'COII':'COX2',
'COIII':'COX3',
'COXI':'COX1',
'COXII':'COX2',
'COXIII':'COX3',
'cytb':'CYTB',
'CYTOCHROME B':'CYTB',
'ATPASE 9':'ATP9',
'ATPASE9':'ATP9',
'ATPASE9':'ATP9',
'ATPASE 6':'ATP6',
'ATPASE6':'ATP6',
'ATP SYNTHASE F0 SUBUNIT 6':'ATP6',
'atp9':'ATP9',
'atp6':'ATP6',
'atp8':'ATP8',
'atp 8':'ATP8',
'ATPASE 8':'ATP8',
'ATP 8':'ATP8',
'cytB':'CYTB',
'cob':'CYTB',
'COB':'CYTB',
'CYT B':'CYTB',
'CYT B':'CYTB',
'NADH1':'ND1',
'NADH DEHYDROGENASE SUBUNIT 1':'ND1',
'NADH2':'ND2',
'NADH DEHYDROGENASE SUBUNIT 2':'ND2',
'NADH3':'ND3',
'NADH DEHYDROGENASE SUBUNIT 3':'ND3',
'NADH4':'ND4',
'NADH DEHYDROGENASE SUBUNIT 4':'ND4',
'NADH4L':'ND4L',
'NADH DEHYDROGENASE SUBUNIT 4L':'ND4L',
'NADH5':'ND5',
'NADH DEHYDROGENASE SUBUNIT 5':'ND5',
'NADH6':'ND6',
'NADH DEHYDROGENASE SUBUNIT 6':'ND6',
'NAD1':'ND1',
'NAD2':'ND2',
'NAD3':'ND3',
'NAD4':'ND4',
'NAD4L':'ND4L',
'NAD5':'ND5',
'NAD6':'ND6',
}

def clean_Gene(gene):
	name=gene.upper()
	if name in clean_dir:
		name=clean_dir[name]
	else:
		pass
	return name
# --------------------------part1---------------------------
def getGeneDic(cdsfile):
	'''
	In this section, a dictionary which I call 'ncdic' like below was generated:
	{'NC_026885': {'ATP8': 'ATTCCACAAATAGCC...',},}
	maybe you can use 'print ncdic' to see it
	'''
	dirlst = os.listdir(cdsfile)
	ncdic = {}
	for i in dirlst:
		species = i.split('.')[0]
		fa = cdsfile+'/'+i
		record = SeqIO.parse(fa, 'fasta')
		genedic = {}
		for j in record:
			gene = re.findall(r'gene=(.*?)]', j.description)
			genedic[clean_Gene(gene[0])] = j.seq
		ncdic[species] = genedic
		# print ncdic
	return ncdic

def check(cdsfile):
	'''
	This funtion used to check the input dir and cds file
	'''
	dirlst = os.listdir(cdsfile)
	genelen = []
	for i in dirlst:
		if re.search(r'~', i):
			os.remove(cdsfile+'/'+i)
			continue
						
		fa = cdsfile+'/'+i
		record = SeqIO.parse(fa, 'fasta')
		genelst = []
		for j in record:
			if re.search('gene=', j.description):
				gene = re.findall(r'gene=(.*?)]', j.description)
				genelst.append(clean_Gene(gene[0]))
			else:
				print '-- Format is abnormal: ' + i + ' without "gene="! \n'
				exit(1)

		for t in genelst:
			if genelst.count(t) > 1:
				print '-- Gene in ' + i + ' has duplication! \n'
				exit(1)

		genelen.append(len(genelst))
	if len(set(genelen)) != 1:
		print '-- There are different gene nums, please check! \n'
		exit(1)	

def getGenList(dic):
	'''
	This function used to get a list like below:
	['ATP6', 'COX3', ]
	'''
	for k, v in dic.items():
		genelst = v.keys()
		break
	# print genelst
	return genelst

def getGenFile(cdsfile, dic, lst):
	'''
	This function used to get a series of gene file in cds dir,
	such as, ATP6.fa COX2.fa and so on 
	NOTE: This function invoked function clean() to remove stop codon
	'''
	for i in lst:
		genefile = cdsfile+'/'+i+'.fa'
		ifile = open(genefile, 'w')
		for k, v in dic.items():
			for vk in v.keys():
				if vk == i:
					sequence = clean(str(dic[k][vk]))
					print >> ifile, '>%s--%s\n%s' %(k, vk, sequence)
		ifile.close()
		
		cmd="muscle3.8.31_i86linux64 -in %s -out %s.ms" % (genefile, genefile)
		os.system(cmd)
		
def clean(seq):
	t = len(seq)
	s = t-3 if t%3==0 else t-(t%3)
	return seq[:s]
		
# ----------------------------part2------------------------------
def geneDic(cdsfile, lst):
	'''
	This function used to get a dictionary which I call genedic like below:
	{'ND4': {'NC_026885': 'ATGTTAAAGTT..', }, }
	maybe you can use 'print genedic' to see it 
	'''
	genedic = {}
	for i in lst:
		msname = cdsfile +'/'+ i +'.fa.ms'
		msfile = SeqIO.parse(msname, 'fasta')
		ncdic = {}
		for j in msfile:
			nc = j.description.split('--')[0]
			ncdic[nc] = str(j.seq)
		genedic[i] = ncdic
	# print genedic
	return genedic

def getSpeciesList(dic):
	'''
	This function used to get a NC number list from genedic, like below:
	['NC_016015', 'NC_026885', ]
	'''
	for k, v in dic.items():
		nclst = v.keys()
		break
	# print nclst
	return nclst

def getResult(dic, splst, lst):
	'''
	This is the end of this program, used to generate result.fa  
	'''
	outfile = open('result.fasta', 'w')
	for i in splst:
		print >> outfile, '>%s' %i
		for j in lst:
			print >> outfile, dic[j][i].strip(),
		print >> outfile, '\n' 
	outfile.close()

# ----------------------------part3------------------------------
def main(cdsfile):
	check(cdsfile)
	ncdic = getGeneDic(cdsfile)
	lst = getGenList(ncdic)
	getGenFile(cdsfile, ncdic, lst)
	
	genedic = geneDic(cdsfile, lst)
	splst = getSpeciesList(genedic)
	getResult(genedic, splst, lst)
	
if __name__=='__main__':
	if len(sys.argv) != 2:
		print 'Usage: <python> <script> <cdsdir>'
		print '<cdsdir> is a dir, which contains the cdsfile of each species'
		exit(1)
	else:
		scgene, cdsfile = sys.argv
		main(cdsfile)

******************************************************************

kaks_stop.py文件：

#! /usr/bin/python 

import sys
from Bio import SeqIO
for i in SeqIO.parse(sys.argv[1],'fasta'):
	length=len(i.seq)
	a=divmod(length,3)
	if a[1] == 0:
		print '>%s\n%s' % (i.id,i.seq[:-3])
	if a[1] != 0:
		print '>%s\n%s' % (i.id,i.seq[:-a[1]])

********************************************************************

kaks_R.py文件：

#! /usr/bin/python 

import sys
import os
import re
def main(fa):
    out=open("%s.R" % (fa),'w')
    out.write("library(\"seqinr\")\nfa <- read.alignment(file=\"%s\",format=\"fasta\")\nresult <- kaks(fa)\nresult" % (fa))
    out.close()
    os.system("Rscript %s.R > %s.txt" % (fa,fa))
    kaks=open("%s.txt" % (fa))
    ka=''
    ks=''
    ka_result={}
    ks_result={}
    for i in kaks:
        line=i.strip().split()
        if ka=='begin':
            if re.search("^ ",i):
                ka_dic={}
                for num in range(0,len(line)):
                    ka_dic[num]=line[num]
            elif len(i.strip()) != 0 :
                access=line[0]
                for num in range(0,len(line[1:])):
                    name="%s-%s" % (ka_dic[num],access)
                    new_ka=line[num+1]
                    if float(line[num+1]) > 9:
                        new_ka="NA"
                    ka_result[name]=new_ka
        if ks =='begin':
            if re.search("^ ",i):
                ks_dic={}
                for num in range(0,len(line)):
                    ks_dic[num]=line[num]
            elif len(i.strip()) != 0 :
                access=line[0]
                for num in range(0,len(line[1:])):
                    name="%s-%s" % (ks_dic[num],access)
                    new_ks=line[num+1]
                    if float(line[num+1]) > 9:
                        new_ks='NA'
                    ks_result[name]=new_ks
        if i.strip() =="$ka":
            ka='begin'
        if i.strip() =="$ks":
            ks='begin'
        if len(i.strip())==0:
            ka='finish'
            ks='finish'
    out1=open("%s.xls" % (fa),'w')
    out1.write("ID\tka\tks\tkaks\n")
    for i,k in ka_result.items():
        ka=float(k)
        ks=float(ks_result[i])
        if ka == "NA" or ks == "NA" or ka == 0 or ks == 0 :
            kaks=0
        if ka != "NA" and ks != "NA" and ka != 0 and ks != 0 :
           kaks=round(ka/ks,3)
        out1.write("%s\t%s\t%s\t%s\n" % (i,ka,ks,kaks))
    out1.close()
    out2=open("%s_draw.R" % (fa),'w')
    out2.write("library(\"ggplot2\")\nfile <- read.table(\"%s.xls\",header=T)\npdf(file=\"%s.pdf\")\nggplot(file,aes(x=ID,y=kaks,fill=ID),y=\"Ka/Ks\") + geom_bar(stat=\"identity\",width=0.4) + geom_text(aes(label=kaks),vjust=-1.0) + theme(axis.text.x = element_text(size = 5,angle=45,hjust=1))"% (fa,fa))
    out2.close()
    os.system("Rscript %s_draw.R" % (fa))
main(sys.argv[1])


*********************************************************************

kaks_sort.py文件：
#! /use/bin/python 

import sys

def sort_kaks(infile,outfile):
	inf=open(infile,'r')
	outf=open(outfile,'w')
	dist={}
	for i in inf:
		a=i.strip().split("\t")
		if a[0]=="Seq1":
			pass
		else:
			b=sorted([a[0].split(".")[0],a[1].split(".")[0]])
			dist['\t'.join(b)]=[a[2],a[3],a[4]]
	for x in sorted(dist.keys()):
		outf.write("%s\t%s\t%s\n" % (infile.split(".")[0],x,'\t'.join(dist[x])))
sort_kaks(sys.argv[1],sys.argv[2])


****************************************************************


6.CCT（如物种太多，要把backboneRadius的值调大，并且把legendFontSize的值调小一点,CDS和DNA的图的物种排列顺序不一定一样）
打开装有CCT的虚拟机上的linux,新建一个文件夹,在里面运行build_blast_atlas.sh -p $1（文件夹名字），vim draw.sh 写下如下内容:
build_blast_atlas.sh -p $1 --map_size x-large -b 50  --custom "backboneRadius=3000 tick_density=0.15 labelFontSize=100 at_skew=T legendFontSize=85"  其中$1是文件夹名字。
把目标物种的gb文件放到$1里面的reference_genome，其他物种放到compare_genome中。
运行 sh draw.sh   #文件的名字不能有空格。
Linux 设置共享文件夹：
$sudo mount -t vboxsf share1 /mnt/share2
share1是本机在设置里面共享的文件夹名字，如下面的2huangjingchuan ,share2是虚拟机呢/mnt/目录下，你创建的目录。

目标图片在map of cds 和map of dna 中。

******************************************************************

7. 扩SNP
7.1 以集群上某处作为工作目录
mkdir 01.fq
将PE reads链接到01.fq中；
将Gentools/V2tools中的整个SNP文件夹复制到工作目录下，并重命名为02.snp
将基因组序列文件（即input中的XX.fsa文件。为了让后续分析不出意外的错，尽量简洁重命名该文件，eg F187.fsa）和genebank文件（即Genbank中*.genbank.txt文件）拷贝到02.snp中
cd 02.snp
sh 01.align.sh XX.fsa ../01.fq/*.fastq.gz > 01.log 2>&1
# 这一步是将genome remap回reads，再用samtools找SNP，最终生成result.raw.vcf
# 其中XX.fsa是genome文件

7.2 运行vcfpass.py进行过滤
python vcfpass.py -d 0.1 -t all result.raw.vcf > result.pass.vcf
# 其中0.1这个值可以调节，这表示某位点发生的变异数占总深度的比率

7.3 运行anote2A.py生成mutant文件
python anoteSNP.py XX_genbank.txt result.pass.vcf > result.mutant.xls
# 运行完生成的result.mutant.xls虽然后缀为xls但依然是文本文件需要保存到Excel中并调整格式
新的call snp和indel的方法：
比对完后得到的bam文件，samtools mpileup -f example.fasta example.bam > example.mpileup
Python call_snp_and_indel.py example.mpileup example.snp.xls example.indel.xls
如过是两个样品以上：ls *.snp.xls |while read a ;do printf “$a:\n” ;cat $a ;printf “\n”;done > all.snp.txt
ls *.indel.xls |while read a ;do printf “$a:\n” ;cat $a ;printf “\n”;done > all.indel.txt
python reorder.py all.snp.txt > all.snp.xls
python reorder.py all.indel.txt > all.indel.xls
python snp_gb.py all.snp.xls example.gb > snp.result.xls#注意，需要read_GB.py在当前目录，如果多于或者少于两个样品需要更改程序。
python indel_gb.py all.indel.xls example.gb > indel.result.xls#注意：需要read_GB.py在当前目录,并且需要打开indel.result.xls,查看哪些是在CDS上（除了orf），把改变的氨基酸填上。

*******************************************************************8. 基因组SSR分析
8.1 以集群上某处作为工作目录
mkdir 01.fq
将PE reads链接到01.fq中
将Gentools/V2tools中的整个SSR文件夹复制到工作目录下，并重命名为02.ssr
再将基因组序列文件XX.fsa和genbank文件XX_genbank.txt复制到02.ssr中，执行
perl misa.pl XX.fsa
# XX.fsa是基因组序列文件
# perl misa.pl可以看到更详见的说明
# 运行完生成一个XX.fsa.misa的文件，

8.2 再运行
python anoteSSR.py XX_genbank.txt XX.fsa.misa
# 这一步是对上步生成的misa文件进行整理
# XX_genbank.txt是目标物种的genbank文件，XX.fsa.misa是上步生成的misa文件
# 这一步调用了read_GB.py

8.3 如果要设引物，可以参照俊哥附加的整体流程文件READ.pip.txt（已经在工作目录下）

10.重复序列分析(不含misc_match)
以集群上某处为工作目录，将基因组序列文件XX.fsa文件复制到工作目录下，同时将Gentools/V2tools/Repeat中的repeat2xls.py复制到工作目录下：
repeat-match -n 30 XX.fsa > repeat.list
python repeat2xls.py XX.fsa repeat.list out.xls outr.xls
# 生成out.xls和outr.xls分别是正向和反向重复的结果文件
# 同样，这里生成的xls也只是后缀为xls的文本文件，需要整理到Excel中，调整格式

11.long_repeat（含misc_match） 
在这个网站上完成http://bibiserv.techfak.uni-bielefeld.de/reputer/)
把结果保存为txt，用xls打开，然后用python new_repeat.py txt genbank out (注意，需要repeat文件夹内的read_GB.pyc文件才能正常运行,起始的位置是和程序一样，起始为0,需要把起始位置全部加1)
12. 修改报告
需要修改的地方有：
a. 物种名，替换所有‘华鲮’为目标物种名
b. 报告日期（首页）




13.p-genetic
（需要每个基因全部核苷酸序列的fa,1-2位的核苷酸序列的fa,3位的核苷酸序列的fa,需计算全部基因，1-2位，3位,all的p-gentic的平均值来作图)注:全部物种的基因需要放在同一个fa
1.打开mega6,File→open A file→导入fa文件，出来一个弹窗点击Align，用muscle比对，点Date→save session,保存。
2.在mega6中打开上一步保存的文件，出来一个弹窗点Analyze,在点击Analysis→distance→compute pairwise distances .. ,点yes,出来一个弹窗，在variance estimation method 中选Bootstrap method ,NO.of Bootstrap method 选1000,Model/Method 中选p-distance.点compute。点击Average→overall ,出来一个值即是p-genetic 的值(一般不会大于1,如大于1,需检查一遍是否有错)

****************************************************************

15.GC-skew
一，多个物种时：
把需要做的物种的GB文件放到同一个文件夹gb下，运行 
Python static.py gb/ 
注意：出来一个Gene Information.xls的文件，打开文件，如发现有空的或者长度与其他物种相差较大时，应打开此物种的GB文件检查，并相应的修改程序（主要是rRNA，除了鱼类，其他物种没有Control region)。
二，只做一个物种时：
asn2all -i *.sqn -o *.nucl -f d
Python AT.py *.nucl > GC-skew.xls
用LibreOffice Calc打开GC-skew.xls，制作成skew2.png的图表

static.py文件：
 # -*- coding: utf-8 -*-
'''''''''''''''''''''''''''''''''''''''''''''''''''''''''
last edit: 2015.10.9
readme: please mkdir fisrt, use python *.py filename/
本程序可算出genbank中的ATskew和GCskew
可修改：如果进来是单个文件怎么样....
'''''''''''''''''''''''''''''''''''''''''''''''''''''''''
from Bio import SeqIO
from xlwt import Workbook,XFStyle,Borders
import xlwt
import sys
import os
import re

script,filename=sys.argv
def getInfo():
        name=[]
        length=[]
        ATpercent=[]
        GCskew=[]
        ATskew=[]
        aalen=[]
        cdsat=[]
        rrnllen=[]
        rrnlat=[]
        rrnslen=[]
        rrnsat=[]
        conlen=[]
        conat=[]
        trnalen=[]
        trnaat=[]
        thirdat=[]
        fname=[]
        accnum=[]

        fname=os.listdir(filename)[:]
        for gbf in fname:
                cdsseq=''
                rrnlseq=''
                rrnsseq=''
                trnaseq=''
                conseq=''
                third=''
		trans=''
                try:
                        gbfile=SeqIO.read(filename+'/'+gbf,'gb')
                except IOError:
                        print gbf,'is not a normal genbank file!'

                #2015.11.6 add +' '+re.findall(r'\w+',gbfile.description)[1]
                spacename=re.findall(r'\w+',gbfile.description)[0]+' '+re.findall(r'\w+',gbfile.description)[1]
                gbid=gbfile.id if (gbfile.id).startswith('N') else ' '
                name.append(spacename)
                accnum.append(gbid)
                wholeseq=str(gbfile.seq)
                length.append(len(wholeseq))
                at=getCont(wholeseq)[0]
                ATpercent.append(at)
                GCskew.append(getCont(wholeseq)[1])
                ATskew.append(getCont(wholeseq)[2])

                for feature in gbfile.features:
                        subseq=''
                        figure=re.findall(r'\d+',str(feature.location))

                        for i in range(len(figure)/2):
                                subseq+=wholeseq[int(figure[i*2]):int(figure[i*2+1])]

                        if feature.type=='CDS':
                                cdsseq+=subseq
				trans+=feature.qualifiers['translation'][0]

			# 2015.11.06 add product/s-rRNA and product/16S
                        if (feature.qualifiers.has_key('gene') and feature.qualifiers['gene'][0].find('rnl')!=-1)\
                           or (feature.qualifiers.has_key('product') and feature.qualifiers['product'][0]=='l-rRNA')\
                           or (feature.qualifiers.has_key('product') and re.search(r'16S',feature.qualifiers['product'][0])):
                               rrnlseq+=subseq

                        if (feature.qualifiers.has_key('gene') and feature.qualifiers['gene'][0].find('rns')!=-1)\
                           or (feature.qualifiers.has_key('product') and feature.qualifiers['product'][0]=='s-rRNA')\
                               or (feature.qualifiers.has_key('product') and re.search(r'12S',feature.qualifiers['product'][0])):
                               rrnsseq+=subseq

                        if feature.type=='tRNA':
                                trnaseq+=subseq

                        if (feature.qualifiers.has_key('note') and re.search(r'[Cc]ontrol region',feature.qualifiers['note'][0]))\
                           or (feature.qualifiers.has_key('product') and re.search(r'[Cc]ontrol region',feature.qualifiers['product'][0]))\
                           or feature.type=='D-loop':
                                conseq+=subseq

                for j in range(len(cdsseq)/3):
                        third+=cdsseq[j*3+2]

                try:
                        aalen.append(len(trans))
                        cdsat.append(getCont(cdsseq)[0])
                        thirdat.append(getCont(third)[0])
                except ZeroDivisionError:
                        print spacename,'cannot find CDS!'
                        cdsat.append(0)
                        thirdat.append(0)
                try:
                        rrnllen.append(len(rrnlseq))
                        rrnlat.append(getCont(rrnlseq)[0])
                except ZeroDivisionError:
                        print spacename,'cannot find rrnL!'
                        rrnlat.append(0)
                try:
                        rrnslen.append(len(rrnsseq))
                        rrnsat.append(getCont(rrnsseq)[0])
                except ZeroDivisionError:
                        print spacename,'cannot find rrnL!'
                        rrnsat.append(0)

                try:
                        trnalen.append(len(trnaseq))
                        trnaat.append(getCont(trnaseq)[0])
                except ZeroDivisionError:
                        print spacename,'cannot find tRNA!'
                        trnaat.append(0)

                try:
                        conlen.append(len(conseq))
                        conat.append(getCont(conseq)[0])
                except ZeroDivisionError:
                        print spacename,gbid,'cannot find Control region!'
                        conat.append(0)

        workbook=xlwt.Workbook()
        sheet1=workbook.add_sheet('static for skew',cell_overwrite_ok=True)
	# 2015.11.9 change GC-skew% >>> GC-skew
        subtitle=['AT%','GC-skew','AT-skew','Length(aa)','AT%(all)','AT%(3rd)',
                  'Length(bp)','AT%','Length(bp)','AT%','Length(bp)','AT%',
                  'Length(bp)','AT%']
        for i,tit in enumerate(subtitle):
                sheet1.write(1,i+3,tit)

        sheet1.write_merge(0,1,0,0,'Speics')
        sheet1.write_merge(0,1,1,1,'Accession number')
        sheet1.write_merge(0,1,2,2,'Length(bp)')
        sheet1.write_merge(0,0,3,5,'Entire genome')
        sheet1.write_merge(0,0,6,8,'Protein-coding gene')
        sheet1.write_merge(0,0,9,10,'rrnL')
        sheet1.write_merge(0,0,11,12,'rrnS')
        sheet1.write_merge(0,0,13,14,'tRNAs')
        sheet1.write_merge(0,0,15,16,'Control region')

        for j in range(len(name)):
                try:
                        sheet1.write(j+2,0,name[j])
                        sheet1.write(j+2,1,accnum[j])
                        sheet1.write(j+2,2,length[j])
                        sheet1.write(j+2,3,ATpercent[j])
                        sheet1.write(j+2,4,GCskew[j])
                        sheet1.write(j+2,5,ATskew[j])
                        sheet1.write(j+2,6,aalen[j])
                        sheet1.write(j+2,7,cdsat[j])
                        sheet1.write(j+2,8,thirdat[j])
                        sheet1.write(j+2,9,rrnllen[j])
                        sheet1.write(j+2,10,rrnlat[j])
                        sheet1.write(j+2,11,rrnslen[j])
                        sheet1.write(j+2,12,rrnsat[j])
                        sheet1.write(j+2,13,trnalen[j])
                        sheet1.write(j+2,14,trnaat[j])
                        sheet1.write(j+2,15,conlen[j])
                        sheet1.write(j+2,16,conat[j])
                except IndexError:
                        pass

                workbook.save('Gene Information.xls')

def getCont(seq):
        a=seq.count('A')
        t=seq.count('T')
        c=seq.count('C')
        g=seq.count('G')
        n=seq.count('N')
        atcgn=a+t+c+g+n
	# 2015.11.13师姐说AT-skew和GC-skew保留4位小数
        GCskew=round(float(g-c)/(g+c),4)
        ATskew=round(float(a-t)/(a+t),4)
        AT=round(float(a+t)/atcgn*100,2)
        return AT,GCskew,ATskew

def main():
        if True:
                getInfo()

if __name__=='__main__':
        main()



*****************************************************************

16.内含子分析
挑选出含有intron的物种的genbank文件放在同一个文件夹内，运行：
ls *.gb | while read a ;do python intron.py $a $a.out ;done 
cat *.out > intron.xls 
打开文件，把intron的类型全部换成GI,GII或者IA1,IB2之类的名称。一些难以分辨的就写unknow.
目标物种的内含子类型就按照intron_vs_gene (CRW数据库).xlsx中对应基因出现最多次数的种类来分类，要看清楚线粒体或者叶绿体。

17.基因基因情况的统计：
在v2中的cpDNA_comparsion中，运行python cpDNA_comparsion gb/ out mit/chl 出来的结果中，intron和intron ORF要自己去genbank中数一下，因为genbank的写发不一，所以比较难统计。如rRNA的数量异常，也需要去查看一下

18.近缘物种基因统计。
首先把要分析的序列的文件下载下来:asn2all -r -A accsion -v accsion.cds -f d ,
运行V2中的gene_count里的程序(分线粒体和叶绿体），python chl_gene_count target.cds cds/ out ,用excel 打开out ,最后那几个是近缘物种没有的，需要另外统计，“-”代表有，‘*’代表没有。

19把tRNA二级结构的分割并做成pdf，png，tif
把V1生成的二级结构的文档复制到新建的文件夹，然后python split_tRNA.py tRNA.txt；就会生成很多单独的tRNA的txt。Ls *.txt |while read a ;do enscript -B -p $a.ps $a ;done
Ls *.txt |while read a ;do ps2pdf $a.ps $a.pdf ;done .

20.维恩图（Venn diagram）
网站http://bioinfogp.cnb.csic.es/tools/venny/index.html
在每个框内写入基因的名字即可

venn.py文件：
#!/usr/bin/python
#-*- coding:utf-8 -*-
import os
import glob

rs=open('r.R','w+')
rs.write('''#install.packages("VennDiagram")
library(VennDiagram)
library(grid)
library(futile.logger)
venn.diagram(x=list(
''')

import os
import glob
all=[]
flist=glob.glob('*.list')
#print flist
for i in flist:
#    print i
    vlist=i.strip().split('.')[0]
#    print n
    for v in vlist:
        fc=open(i,'r').readlines()
    #print "%s = %s"%(vlist,fc)
    rs.write("%s = %s,"%(vlist,fc))

rs.write('''),
             height = 720, width = 720,resolution =720,
             filename ="venn.tiff",imagetype = "tiff",
             col="white",
''')

colors=["#FF1493", "#FF00FF","#FFFF00","#FFD700","#FFA500","#8B4513","#808080","#0000FF","#00CED1","#00FF7F","#2E8B57","#00FF00","#FAFAD2","#DC143C","#191970","#4169E1","#2F4F4F",",#EEE8AA","#FF6347",",#000000"]                                       
alphas=[0.5, 0.5, 0.5,0.5,0.5,0.5,0.5,0.5,0.5,0.5,0.5,0.5,0.5, 0.5, 0.5,0.5,0.5,0.5,0.5,0.5,0.5,0.5,0.5,0.5]
#linew=[0.5,0.5,0.5,0.5,0.5,0.5,0.5,0.5,0.5,0.5,0.5,0.5]
counter=len(flist)

fill1=colors[0:counter]
rs.write("\t\t\t\t\t\t fill=%s,\n" % fill1)

alpha1=alphas[0:counter]
alpha2=map(str,alpha1)
alpha3=",".join(alpha2)
rs.write("\t\t\t\t\t\t alpha=c(%s)," % alpha3)

#ln1=linew[0:counter]
#ln2=map(str,ln1)
#ln3=",".join(ln2)
#rs.write("\t\t\t\t\t\t lwd=c(%s)," % ln3)

rs.write('''       
             cex=0.1,
			 cat.cex=0.2,cat.pos=0,cat.fontface="bold",cat.fontfamily="serif",
			 reverse=FALSE,
			 rotation.degree =0,
			 lwd=0.5,)
''')

rs.close()
fpr=open('r.R','r')
fp=open('venn.R','w+')
for s in fpr.readlines():
	fp.write(s.replace('[','c(').replace(']',')').replace(',)',')'))
fp.close()
fpr.close()

os.system("rm r.R")

os.system("Rscript venn.R")

*************************************************************

五、与cleanreads比对
1.bowtie
bowtie2-2.2.5比对
samtools查看和排序，生成.bai文件
qsub提交大型机

bowtie.sh文件：
#$ -S /bin/sh
if [ $# -eq 0 ]; then
        echo "Program: Package the sequecing data and statics for mtFish sequencing"
        echo "Usage: $0 <fasta> <fq1> <fq2>"
        exit 1
fi

ref=$1
fq1=$2
fq2=$3
echo '#$ -S /bin/sh' > bow.sh
echo "/hellogene/scgene01/bio/software/bowtie2-2.2.5/bowtie2-build ${ref} long" >> bow.sh
echo "/hellogene/scgene01/bio/software/bowtie2-2.2.5/bowtie2 -x long -1  ${fq1} -2 ${fq2} -p 25 | /hellogene/scgene01/bio/bin/samtools view -SbF 0x804 - | /hellogene/scgene01/bio/bin/samtools sort -@ 25 - > long.srt.bam " >> bow.sh
echo  '/hellogene/scgene01/bio/bin/samtools index long.srt.bam'  >> bow.sh
qsub -cwd -l vf=20g bow.sh

2.bwa比对
bwa
samtools查看和排序，生成.bai文件
qsub提交大型机

#$ -S /bin/sh
if [ $# -eq 0 ]; then
        echo "Program: Package the sequecing data and statics for mtFish sequencing"
        echo "Usage: $0 <genome.fasta> <fq1> <fq2>"
        exit 1
fi

ref=$1
fq1=$2
fq2=$3
echo '#$ -S /bin/sh' > remap.sh
echo "/hellogene/scgene01/bio/bin/bwa index ${ref}" >> remap.sh
echo "/hellogene/scgene01/bio/bin/bwa mem -t 14 ${ref}  ${fq1} ${fq2} | /hellogene/scgene01/bio/bin/bwa view -SbF 0x804 - | /hellogene/scgene01/bio/bin/samtools sort -@ 8 - > genome.map.bam" >> remap.sh
echo  '/hellogene/scgene01/bio/bin/samtools index genome.map.bam'  >> remap.sh
qsub -cwd -l vf=12g remap.sh


=================================================================

六、分子钟
网站http://www.evolgenius.info/evolview/#mytrees/140/time
参考年代图：https://baike.baidu.com/pic/%E5%9C%B0%E8%B4%A8%E5%B9%B4%E4%BB%A3%E8%A1%A8/3725774/0/c2cec3fdfc039245847aa73a8f94a4c27d1e2533?fr=lemma&ct=single#aid=0&pic=c8177f3e6709c93d98fff54a973df8dcd100547a

注意，#行的内容别复制进去，网页无法识别#是注释
!TimeLine	TotalTime=140,TimeUnit=Millions of Years
#设置时间线，刻度总长度
!TimeLineAxis
#设置时间轴
Pos=Bottom,
#刻度position位置在底部，有Top，Left，Bottom，Right
Ticks=20,10,5,
#最小刻度5，中等刻度10，大刻度为20，即(大)0_(小)5_(中)10_(小)15_(大)20
TickLabelStyle=10,black,0,0,
#刻度标签：字体大小，字体颜色，是否斜体，是否粗体
Grid=3
!TimeLineStrips
#设置时间线分割
op=1,
#不透明度0-1
Strips=-10,2.6,23.0,65.5,145.5,
#纵向刻度分割线所在刻度，如果树的最大年为100，而设置的总刻度为140，QUATER NARY:2.6,NEOGENE:23.0,PALEOGENE:65.5,CRETACEOUS:145.5是标准时间，需要100/140*2.6=计算，当然最合适的是把总刻度也变为100，这时这几个刻度就等于标准时间
StripColors=#FFF68F,#FFB90F,#F4A460,#7CCD7C,
#四个世纪所区别的颜色设置
StripLabelStyle=12,black,0,0,
StripLabelPos=Bottom,
#分割标签的位置
StripMarginPx=1
#分割线的宽度，1比较合适

===================================================================

七、SNP位点注释
结果：
Mutation type	Position	Samples		Gene type	Gene	Codon	Amino acid
		WYL1	WYL3				
SNP	1296	A	G	rRNA	16S-rRNA		
SNP	7354	C	T	CDS	COXII	TCC->TCT	Ser->Ser
SNP	8502	A	G	CDS	ATP6	GAA->GAG	Glu->Glu
SNP	12774	G	A	CDS	ND5	GCC->ACC	Ala->Thr
SNP	14503	T	C	CDS	Cytb	TTA->CTA	Leu->Leu


snp.py
#! /usr/bin/python
# -*- coding: UTF-8 -*-
import sys
from Bio import SeqIO
#程序是两条序列比对后，分别把他们取出再放入fa1和fa2两个文件，然后python SNP.py fa1 fa2 fa1.tbl
#如果有3条序列，可以1和2比后用1的fa1.tbl作为输入，也可以2和3比后用1的fa1.tbl作为输入
#运行完会报错，但是补影响结果，
#fa1.tbl是fa1的注释文件，注意，注释文件最好是和我们注释的样式一模一样，每一个基因都应该是：
#1    100    gene
#            gene    cox1
#1    100    CDS
#            product    XXXXX
#输出结果的位置是比对序列的位置，不是原始序列的位置，所以需要更改，即fa1出现‘-’时。大于这个位置的都需要减1
codon={ "TTT":"Phe","TTC":"Phe","TTA":"Leu","TTG":"Leu",
    "TCT":"Ser","TCC":"Ser","TCA":"Ser","TCG":"Ser",
    "TAT":"Tyr","TAC":"Tyr","TAA":"Stp","TAG":"Stp",
    "TGT":"Cys","TGC":"Cys","TGA":"Trp","TGG":"Trp",
    "CTT":"Leu","CTC":"Leu","CTA":"Leu","CTG":"Leu",
    "CCT":"Pro","CCC":"Pro","CCA":"Pro","CCG":"Pro",
    "CAT":"His","CAC":"His","CAA":"Gln","CAG":"Gln",
    "CGT":"Arg","CGC":"Arg","CGA":"Arg","CGG":"Arg",
    "ATT":"Ile","ATC":"Ile","ATA":"Met","ATG":"Met",
    "ACT":"Thr","ACC":"Thr","ACA":"Thr","ACG":"Thr",
    "AAT":"Asn","AAC":"Asn","AAA":"Lys","AAG":"Lys",
    "AGT":"Ser","AGC":"Ser","AGA":"Stp","AGG":"Stp",
    "GTT":"Val","GTC":"Val","GTA":"Val","GTG":"Val",
    "GCT":"Ala","GCC":"Ala","GCA":"Ala","GCG":"Ala",
    "GAT":"Asp","GAC":"Asp","GAA":"Glu","GAG":"Glu",
    "GGT":"Gly","GGC":"Gly","GGA":"Gly","GGG":"Gly"}
def reverse(seq):
    Base={'A':'T','C':'G','G':'C','T':'A'}
    a=[]
    for i in seq :
        a.append(Base[i])
    result=''.join(a)[::-1]
    return result
def main(fa1,fa2,tbl):
    tbl_dic={}
    for i in open(tbl,'r'):
        if len(i.strip()) == 0:
            continue
        if i.strip()[0] !='>':
            line=i.strip().split()
            if line[0]=="gene":
                gene=line[1]
            if len(line) > 2 :
                if line[-1] == "CDS" or line[-1] == "tRNA" or line[-1] =="rRNA" or line[-1] == "D-loop" :
                    if int(line[0]) < int(line[1]) :
                        start =int(line[0])
                        end= int(line[1])
                        strang='+'
                    if int(line[0]) > int(line[1]):
                        start=int(line[1])
                        end=int(line[0])
                        strang='-'
                    type =line[-1]
            if line[0] == 'product':
                tbl_dic[gene]=[start,end,strang,type]
    fa1_seq=SeqIO.read(fa1,'fasta').seq
    fa2_seq=SeqIO.read(fa2,'fasta').seq
    length=len(fa1_seq)
    for i in range(0,length):
        if fa1_seq[i] != fa2_seq[i]:
            if fa1_seq[i] =='-':
                for k,l in tbl_dic.items():
                    if l[0] > i+1 :
                        tbl_dic[k][0]=l[0]+1
                    if l[1] > i+1 :
                        tbl_dic[k][1]=l[1]+1
            size_type='IGS'
            size_gene=' '
            size_strang=' '
            for k,l in tbl_dic.items():
                if  l[0]< i+1 < l[1]:
                    size_gene=k
                    size_type=l[-1]
                    break
            if size_type=='CDS':
                gene_info=tbl_dic[size_gene]
                if gene_info[2] == '+':
                    size=i+2-gene_info[0] 
                    num=divmod(size,3)
                    if num[1] == 0:
                        size_codon1=fa1_seq[i-2:i+1]
                        size_codon2=fa2_seq[i-2:i+1]
                    if num[1] == 1:
                        size_codon1=fa1_seq[i:i+3]
                        size_codon2=fa2_seq[i:i+3]
                    if num[1] == 2 :
                        size_codon1=fa1_seq[i-1:i+2]
                        size_codon2=fa2_seq[i-1:i+2]
                    print 'SNP\t%s\t%s->%s\t%s\t%s\t%s->%s\t%s->%s' % (i+1,fa1_seq[i],fa2_seq[i],gene_info[-1],size_gene,size_codon1,size_codon2,codon[size_codon1],codon[size_codon2])
                if gene_info[2] == '-' :
                    size=abs(i+2-gene_info[1])
                    num=divmod(size,3)
                    if num[1] == 0:
                        size_codon1=reverse(fa1_seq[i-2:i+1])
                        size_codon2=reverse(fa2_seq[i-2:i+1])
                        if num[1] == 1:
                                size_codon1=reverse(fa1_seq[i:i+3])
                                size_codon2=reverse(fa2_seq[i:i+3])
                        if num[1] == 2 :
                                size_codon1=reverse(fa1_seq[i-2:i+1])
                                size_codon2=reverse(fa2_seq[i-2:i+1])
                    print 'SNP\t%s\t%s->%s\t%s\t%s\t%s->%s\t%s->%s' % (i+1,fa1_seq[i],fa2_seq[i],gene_info[-1],size_gene,size_codon1,size_codon2,codon[size_codon1],codon[size_codon2])
            if size_type != 'CDS':
                print 'SNP\t%s\t%s->%s\t%s\t%s' % (i+1,fa1_seq[i],fa2_seq[i],size_type,size_gene)
main(sys.argv[1],sys.argv[2],sys.argv[3])

========================================================================

简单序列操作
long_seq1.py文件，取反向序列，python long_seq1.py infile outfile
from Bio import SeqIO
import sys

scgene, faf, outf=sys.argv
file=open(faf,'r')
out=open(outf,'w')
for i in SeqIO.parse(file,'fasta'):
	out.write('>%s\n%s\n' % (i.id,i.seq))
out.close()

************************************************************************
long_seq2.py文件
from Bio import SeqIO
from Bio.Seq import Seq
import sys

scgene, faf, outf=sys.argv
file=open(faf,'r')
out=open(outf,'w')
for i in SeqIO.parse(file,'fasta'):
	out.write('>%s\n%s\n' % (i.id,i.seq))
	a=Seq(str(i.seq))
	c=a.reverse_complement()
	out.write('>%s\n%s\n' % (i.id,c))
out.close()

=======================================================================

七、根据物种的taxid及物种名下载其序列
网页：NCBI——全数据库搜索——物种分类（左下边）——基因组（右上边1）点击1即可到达下载页面
taxid	species_name
8897	Chaetura_pelagica
9244	Calypte_anna
175835	Buceros_rhinoceros
279965	Antrostomus_carolinensis
注意：物种名的空格应用下划线替代，否则创建文件夹等步骤会出错

测试是否有转录组和基因组的数据可下载，如果有则显示链接，如果没有则显示[]每个循环即物种间******分隔
trans_proti_ts.py文件

# -*- coding:utf-8 -*-

import urllib
import os
import sys
import re
from bs4 import BeautifulSoup

def main(list_file):
    f=open(list_file,'r')
    for n in f:
        n1=n.strip().split("\t")[0]
        n2=n.strip().split("\t")[1]
        ln="https://www.ncbi.nlm.nih.gov/genome/?term=txid"+n1
        connect = urllib.urlopen(ln)
        url_html=connect.read()
        soup= BeautifulSoup(url_html)
#        os.system("mkdir %s" % n2)
        a1=soup.find_all(href=re.compile("rna.fna.gz"))
        for a11 in a1:
            rna= a11.get("href")
            print rna
#            os.system("wget %s" % rna)
        a2= soup.find_all(href=re.compile("protein.faa.gz"))
        for a22 in a2:
            pro= a22.get("href")
            print pro
#            os.system("wget %s" % pro)
#        os.system("mv *.gz %s" % n2)
        print "******"
main(sys.argv[1])
**********************************************************************

下载对应的转录组和氨基酸序列，每个物种会根据物种名创建文件夹，当下载完该物种序列后会将序列移动到文件夹内，屏幕上会显示正在进行的步骤及下载进度

trans_proti_dl.py文件：
# -*- coding:utf-8 -*-

import urllib
import os
import sys
import re
from bs4 import BeautifulSoup

def main(list_file):
    f=open(list_file,'r')
    for n in f:
        n1=n.strip().split("\t")[0]
        n2=n.strip().split("\t")[1]
        ln="https://www.ncbi.nlm.nih.gov/genome/?term=txid"+n1
        connect = urllib.urlopen(ln)
        url_html=connect.read()
        soup= BeautifulSoup(url_html)
        os.system("mkdir %s" % n2)
        a1=soup.find_all(href=re.compile("rna.fna.gz"))
        for a11 in a1:
            rna= a11.get("href")
            os.system("wget %s" % rna)
        a2= soup.find_all(href=re.compile("protein.faa.gz"))
        for a22 in a2:
            pro= a22.get("href")
            os.system("wget %s" % pro)
        os.system("mv *.gz %s" % n2)
        print "******"
main(sys.argv[1])
****************************************************************************
将网页的下载页数全部显示后，CTRL+s保存网页文件为输入参数，获取，页面的物种名和id，id之后用来下载其它信息文件，拉丁名创建文件夹时注意将空格替换为下划线
#! /usr/bin/python

import urllib
import sys
import re

def main(html_file):
	file=open(html_file,'r')
#	out=open(out_file,'w')
	id=[]
	for i in file:
		if re.search(r'link_uid=',i.strip()):
			id_list=i.split('link_uid=')
			id_list=id_list[1:]
			for i in id_list:
				id.append(i.split("\">")[0])
                ids="\n".join(id)
                print ids
	for i in id:
		url="https://www.ncbi.nlm.nih.gov/assembly?LinkName=genome_assembly&from_uid=%s" % (i)
		refseq_id_list1=[]
		connect=urllib.urlopen(url)
		url_text=connect.read()
		html_file=open('html_file.txt','w')
		html_file.write("%s" % (url_text))
		html_file.close()
		html_file_input=open('html_file.txt','r')
		for j in html_file_input:
			if re.search('Organism: </dt><dd>',j.strip()):
				species_name=j.split('Organism: </dt><dd>')[1].split('(')[0]
                name.append(species_name)
                print name
main(sys.argv[1])


****************************************************************************
根据id和物种名下载genbank，genome,trans,protein,gff文件
# -*- coding:utf-8 -*-

import urllib
import os
import sys
import re
from bs4 import BeautifulSoup

def main(list_file):
    f=open(list_file,'r')
    for n in f:
        n1=n.strip().split("\t")[0]
        n2=n.strip().split("\t")[1]
        ln="https://www.ncbi.nlm.nih.gov/genome/"+n1
        connect = urllib.urlopen(ln)
        url_html=connect.read()
        soup= BeautifulSoup(url_html)
        os.system("mkdir %s" % n2)
        '''
        a1=soup.find_all(href=re.compile("rna.fna.gz"))
        for a11 in a1:
            rna= a11.get("href")
            print rna
            os.system("wget %s" % rna)
        os.system("mv *.gz %s" % n2)
        a2= soup.find_all(href=re.compile("protein.faa.gz"))
        for a22 in a2:
            pro= a22.get("href")
            print pro
            os.system("wget %s" % pro)
        os.system("mv *.gz %s" % n2)
        '''
        a3 = soup.find_all(href=re.compile("genomic.gff.gz"))
        for a33 in a3:
            gff=a33.get("href")
            print gff
            os.system("wget %s" % gff)
        os.system("mv *.gz %s" % n2)
        a4=soup.find_all(href=re.compile("genomic.gbff.gz"))
        for a44 in a4:
            gbk=a44.get("href")
            print gbk
            os.system("wget %s" % gbk)
        os.system("mv *.gz %s" % n2)
        '''
        a5=soup.find_all(href=re.compile("genomic.fna.gz"))
        for a55 in a5:
            gen=a55.get("href")
            print gen
            os.system("wget %s" % gen)
        os.system("mv *.gz %s" % n2)
        '''
        print "******"
main(sys.argv[1])


注意根据需要注释不需要下载的文件，注意注释’‘’号的index位置，在左端会报错
=========================================================================

八、两个表格文件，根据某一列对应相同将其他几列数据添加到另一文件的前面
在excel中可以使用=IFERROR(VLOOKUP($A2,sheet1!$A:$C,MATCH(B$1,sheet1!$1:$1,0)0))

#-*- coding:utf-8 -*-
Orecord = open('record.xls','r')
Orbcl = open('rbcl.xls','r')
#Frbcl=open('frbcl.xls','w')
frbcl=open('frbcl.xls','w')
info=[]
for rowl in Orecord:
    rowl=rowl.strip().split('\t')
    info.append([rowl[0],rowl[1],rowl[2],rowl[3],rowl[4],rowl[5],rowl[6],rowl[7]])
#print info

for line in Orbcl:
    line=line.strip().split('\t')
    for i in info:
        if i[0] in line[3]:
            i.append([line[0],line[1],line[2],line[3],line[4],line[5],line[6],line[7]])
            #print i
            try:
                frbcl.write("%s\t%s\t%s\t%s\t%s\t%s\t%s\t%s\t%s\t%s\t%s\t%s\t%s\t%s\t%s\t%s\n" % (i[0],i[1],i[2],i[3],i[4],i[5],i[6],i[7],i[8][0],i[8][1],i[8][2],i[8][3],i[8][4],i[8][5],i[8][6],i[8][7]))
            except TypeError:
                pass
frbcl.close()
Orbcl.close()
Orecord.close()

==========================================================================

九、ncbi的sra-toolkit使用

sra-toolkit安装：
sudo apt install sra-toolkit
安装后其下包括多个工具

1.下载序列
prefetch -h
#查看帮助
prefetch accessioni_id
#下载sra序列文件
prefetch SRR1553610
#下载的文件默认保存在/home/username/ncbi/public/sra/SRR1553610.sra

常用命令
Data transfer:
# 如果已有下载的文件是否强制下载，默认为非强制
-f  |   --force <value> Force object download. One of: no, yes, all. no [default]: Skip download if the object if found and complete; yes: Download it even if it is found and is complete; all: Ignore lock files (stale locks or if it is currently being downloaded: use at your own risk!).

# 选择下载的方式 ascp 和 http，默认先尝试 ascp，再尝试http
--transport <value> Value one of: ascp (only), http (only), both (first try ascp, fallback to http). Default: both.

# 列举 kart 文件中的 内容，大小
# 你可以把需要下载的项目放入 kart 文件
-l  |   --list  List the contents of a kart file.
-s  |   --list-sizes    List the content of kart file with target file sizes.

# 设置文件的最小尺寸
-N  |   --min-size <size>   Minimum file size to download in KB (inclusive).

# 设置文件的最大尺寸
-X  |   --max-size <size>   Maximum file size to download in KB (exclusive). Default: 20G.

# 排序方式
-o  |   --order <value> Kart prefetch order. One of: kart (in kart order), size (by file size: smallest first). default: size.

2.文件格式转换
fastq-dump -h
#查看帮助
fastq-dump --split-files accession_id
fastq-dump --split-files local_file

fastq-dump --split-files SRR1553610.sraRead 219837 spots for SRR1553610.sra
#从NCBI下下来的数据，双端测序数据是放在一个文件里的，所以需要把它们重新拆解为两个fasta格式的文件

General:
-h  |   --help  Displays ALL options, general usage, and version information.
-V  |   --version   Display the version of the program.
Data formatting:
#分割 paired-end data
--split-files   Dump each read into separate file. Files will receive suffix corresponding to read number.
--split-spot    Split spots into individual reads.

# 只保留fasta，没有质量得分
--fasta <[line width]>  FASTA only, no qualities. Optional line wrap width (set to zero for no wrapping).
-I  |   --readids   Append read id after spot id as 'accession.spot.readid' on defline.
-F  |   --origfmt   Defline contains only original sequence name.
-C  |   --dumpcs <[cskey]>  Formats sequence using color space (default for SOLiD). "cskey" may be specified for translation.
-B  |   --dumpbase  Formats sequence using base space (default for other than SOLiD).
-Q  |   --offset <integer>  Offset to use for ASCII quality scores. Default is 33 ("!").
Filtering:
-N  |   --minSpotId <rowid> Minimum spot id to be dumped. Use with "X" to dump a range.
-X  |   --maxSpotId <rowid> Maximum spot id to be dumped. Use with "N" to dump a range.
-M  |   --minReadLen <len>  Filter by sequence length >= <len>
--skip-technical    Dump only biological reads.
--aligned   Dump only aligned sequences. Aligned datasets only; see sra-stat.
--unaligned Dump only unaligned sequences. Will dump all for unaligned datasets.

# 输出数据
Workflow and piping:
-O  |   --outdir <path> Output directory, default is current working directory ('.').
-Z  |   --stdout    Output to stdout, all split data become joined into single stream.
--gzip  Compress output using gzip.
--bzip2 Compress output using bzip2.

3.查看文件内容和计算
head SRR1553610_1.fastq
#计算含有@SRR的行数
cat *.fastq | grep @SRR | wc -l

4.同时下载多个文件
echo SRR1553607 > sra.ids
echo SRR1553605 >> sra.ids
#--option-file参数下载列表里多个文件
prefetch --option-file sra.ids


5.更方便地下载，使用esearch和efetch
sudo apt install acedb-other
sudo apt install ncbi-entrez-direct

esearch -db sra -query PRJNA257197  | efetch -format runinfo
#查看
esearch -db sra -query PRJNA257197  | efetch -format runinfo > runinfo.txt
cat runinfo.txt|wc -l
927
cat runinfo.txt|grep SRR|wc -l
891
#由于这是一个逗号分隔符的文件，我们需要把分隔符（用以区别不同列的符号）指明给cut程序
cat runinfo.txt | cut -f 1 -d ","
网页版的截图里，你也可以看到，最终结果确实是891个
6.查询当前用户目录下的所占用的空间
du -hs * 
7.在NCBI 网站上的手动操作，获取一个project里的所有SRR ID
首先进入https://www.ncbi.nlm.nih.gov/sra/
输入你要找的这个编号：PRJNA257197 
点击search
会看到很多检索结果 
点击右上角的send to 
选定File，并把Format改为RunInfo
点击Create File就生成了一个SraRunInfo.csv文件了 

=======================================================================

十一、kmer.sh和soap

#$ -S /bin/sh
mkdir K96
SOAPdenovo2-bin-LINUX-generic-r240/SOAPdenovo-127mer all -s lib.conf -o K96/genome -K 96 -p 32
mkdir K127
SOAPdenovo2-bin-LINUX-generic-r240/SOAPdenovo-127mer all -s lib.conf -o K127/genome -K 127 -p 32

========================================================================

十二、大型机使用

1.上传文件
scp ./test.txt miaobenben@192.168.1.5:/home/miaobenben/test

2.下载文件
scp miaobenben@192.168.1.5:/home/miaobenben/test/test.txt ./home/

3.登录节点
ssh miaobenben@192.168.1.5

4.任务管理节点：负责计算任务的投放管理
计算节点：用于任务计算，均为32核及以上CPU，最小内存2G

5.集群上没有~目录，只有/home等

6.任务投放与管理
6.1 投放
qsub -cwd -l vf=8g,p=4 -q prj.q test.sh
-cwd 当前目录下运行，一般需要先新建这个目录
-l vf=8g,p=4 申明需要占用8g内存，4个CPU，内存是使用可以是小数
-q 使用队列，all.q 包含所有节点，rd.q 为rd预留节点，prj.q 为项目分析预留的节点
注意：内存过小任务运算过程可能被自动kill；
p参数应与程序实际使用的线程相符

6.2 任务状态
qstat 当前用户投放情况
qstat -u "*" 查看所有用户任务
qstat -j jobID 查看任务的详细信息
qstat -f 查看任务状态和节点状态

6.3 挂起任务
qhold -j jobID 挂起对应任务
qhold -u user_name 挂起某用户的所有任务

6.4 任务解挂
qrls -j jobID
qrls -u user_name

6.5 任务删除
qdel -j jobID 
qdel jobid
qdel -u user_name

6.6 查看节点状态 qhost

=================================================================

十三、比对和补gap，非鱼类线粒体

1.修改bow.sh文件KQ.fa文件路径及文件名，2个fq.gz的路径及文件名,-p和-@都是CPU数量应与提交任务一致
#$ -S /bin/sh
/hellogene/scgene01/bio/software/bowtie2-2.2.5/bowtie2-build ../KQ.fa long
/hellogene/scgene01/bio/software/bowtie2-2.2.5/bowtie2 -x long -1 ../01.fq/RK80404KQCGJ_DSW62564-V_1.clean.fq.gz -2 ../01.fq/RK80404KQCGJ_DSW62564-V_2.clean.fq.gz -p 25 | /hellogene/scgene01/bio/bin/samtools view -SbF 0x804 - | /hellogene/scgene01/bio/bin/samtools sort -@ 25 - > long.srt.bam 
/hellogene/scgene01/bio/bin/samtools index long.srt.bam


bowtie比对（严格）和remap比对（松散）
bowtie.sh
#$ -S /bin/sh
if [ $# -eq 0 ]; then
        echo "Program: Package the sequecing data and statics for mtFish sequencing"
        echo "Usage: $0 <fasta> <fq1> <fq2>"
        exit 1
fi


ref=$1
fq1=$2
fq2=$3
echo '#$ -S /bin/sh' > bow.sh
echo "/hellogene/scgene01/bio/software/bowtie2-2.2.5/bowtie2-build ${ref} long" >> bow.sh
echo "/hellogene/scgene01/bio/software/bowtie2-2.2.5/bowtie2 -x long -1  ${fq1} -2 ${fq2} -p 25 | /hellogene/scgene01/bio/bin/samtools view -SbF 0x804 - | /hellogene/scgene01/bio/bin/samtools sort -@ 25 - > long.srt.bam " >> bow.sh
echo  '/hellogene/scgene01/bio/bin/samtools index long.srt.bam'  >> bow.sh
qsub -cwd -l vf=20g bow.sh

remap.sh
#$ -S /bin/sh
if [ $# -eq 0 ]; then
        echo "Program: Package the sequecing data and statics for mtFish sequencing"
        echo "Usage: $0 <genome.fasta> <fq1> <fq2>"
        exit 1
fi


ref=$1
fq1=$2
fq2=$3
echo '#$ -S /bin/sh' > remap.sh
echo "/hellogene/scgene01/bio/bin/bwa index ${ref}" >> remap.sh
echo "/hellogene/scgene01/bio/bin/bwa mem -t 14 ${ref}  ${fq1} ${fq2} | /hellogene/scgene01/bio/bin/samtools view -SbF 0x804 - | /hellogene/scgene01/bio/bin/samtools sort -@ 8 - > genome.map.bam" >> remap.sh
echo  '/hellogene/scgene01/bio/bin/samtools index genome.map.bam'  >> remap.sh
qsub -cwd -l vf=12g remap.sh

将bowtie 和remap添加在.bashrc
alias bowtie='/home/miaobenben/sh/bowtie.sh'
alias remap='/home/miaobenben/sh/remap.sh'
如果遇到权限不够时需要在sh所在文件夹下修改sh的权限为777
chmod 777 *.sh

2.提交reads比对任务,产生bam文件
qsub -cwd -l vf=18g,p=25 -q prj.q bow.sh

3.查看bam文件，包括reads覆盖的效果参数
samtools view long.srt.bam|less -S

samtools view long.srt.bam |tail

4.提取reads
python spade.py long.srt.bam KQ.fa ../01.fq/RK80404KQCGJ_DSW62564-V_1.clean.fq.gz ../01.fq/RK80404KQCGJ_DSW62564-V_2.clean.fq.gz 2614600 2620500

5.vim查找string
/string

vim把多个N换成1个N缩小空间
:%s/NNNNNNNNNNNNNNNNNNNNNNNNNNNNN/N/g

6.补reads
在KQ里查找NNN区域并在其前后添加序列，记得先备份
vim打开KQ.fa文件查找NNN，从最后开始补，这样可以同时把文件中所有的NNN区域都补一遍，如果从开头开始会因为增加序列位置改变
在scaffold_long.fa里查找NNN区域前/后的一段序列，并把查到的结果对应前后的序列复制在KQ.fa文件里，此过程同时打开两个终端在同意目录下完成


vim中上键和下键切换行，所以可以将找到后的序列前面或后边部分换行以复制或者替换
由于目标序列有正向和反向同时存在，为了用补好的一小段序列来替换与它重复的序列，同时考虑正反向可以同时搜索并匹配，所有将这段补好的序列用long_seq2.py补好产生一个反向互补的序列在同一个文件中，来同时搜索正反向序列，保证原来序列中正向和反向都可以找到。

7.再次reads比对和查看bam文件，用igv软件查看效果，window软件，需要将.fa .bam .bam.bai三个文件放在同一目录下导入.fa .bam文件查看，.bam.bai文件不可少

======================================================================

十四、从lean.fastq文件开始

1.fastqc质量查看线粒体操作提交
mkdir XXX
cd XXX
cp ../readsQC.py .
mkdir 01.fq
cd 01.fq
ln -s /hellogene/scgene01/rawdata/SCG-80507-250/2.cleandata/s80510SJYL_DSW63405-V/*.gz .
cd ..
python readsQC.py step1 01.fq/*gz	#fastqc，产生相应的表格数据文件查看Q20和Q30的值
python readsQC.py step2 01.fq/*gz	#提交到多线程任务计算，大概需要2小时

************************************************************************
readsQC.py文件

#! /usr/bin/env python
# -*- coding: utf-8 -*-
'''''''''''''''''''''''''''''''''''''''''''''''''''
本程序用于对拿到的reads做前期QC
mkdir 01.fq, 将fq.gz文件链接过来后
python readsQC.py step1 01.fq/*.fq.gz
# 会生成02.qc文件夹，里边放的是初步QC结果
再确定NCBI上是否有比较近的物种，如果有
mkdir 00.ref, 将NCBI上找到的几个近缘物种的fna文件（如XX.fa）放到00.ref中
python readsQC.py step2 -r 00.ref/XX.fa 01.fq/*.fq.gz
如果NCBI没有比较近的近缘物种,先进行组装
python readsQC.py step2 01.fq/*.fq.gz
因为组装程序上抛集群，本程序在此不设监控，待组装完成后
python readsQC.py getscf 03.ass/spadeAss/scaffolds.fasta
Author: Luria
Data: 2016.8.29
'''''''''''''''''''''''''''''''''''''''''''''''''''
import sys
import os
import re
import gzip
import pysam
import argparse
from Bio import SeqIO

ITOOLS = '/hellogene/scgene01/bio/bin/iTools'
FASTQC = '/hellogene/scgene01/bio/software/FastQC/fastqc'
BWA = '/hellogene/scgene01/bio/bin/bwa'
SAMTOOLS = '/hellogene/scgene01/bio/bin/samtools'
SPADES = '/hellogene/scgene01/bio/pip/ass/SPAdes-3.5.0-Linux/bin/spades.py'
PYTHON = '/hellogene/scgene01/opt/bin/python'
ALIGNPATH = '02.qc/align'
ASSPATH = '03.ass'

def info2xls1(baseinfo, replic):
	outfile = open('02.qc/Basicinfo.xls', 'w')
	print >> outfile, '#%s \t%s \t%s \t%s \t%s \t%s \t%s' %('FastqFile',
	'ReadsNum', 'BaseNum', 'GC', 'Q20', 'Q30', 'RepRate')

	#~ Dispose baseinfo.txt file, which contain the information of Q20, Q30, GC and so on
	baseinfo = open(baseinfo, 'r')
	infolist = []
	for i in baseinfo:
		if i.startswith('##'):
			samplename = i.strip().strip('#')
			infolist.append(samplename)
		if i.startswith('#ReadNum'):
			numlist = re.findall(r'\d+', i)
			infolist.append(numlist[0])
			infolist.append(numlist[1])
		if i.startswith('#GC'):
			gc = i.split()[1]
			infolist.append(gc)
		if i.startswith('#BaseQ') and re.search(r'Q20', i):
			Q20 = i.split()[-1]
			infolist.append(Q20)
		if i.startswith('#BaseQ') and re.search(r'Q30', i):
			Q30 = i.split()[-1]
			infolist.append(Q30)

	#~ Dispose Replic.txt file, which contain replication information
	repline = open(replic, 'r').next()
	reprate = repline.strip().split()[-1]
	infolist.append(reprate)
	if len(infolist) != 13:
		print '\033[1;32;41m'
		print '-- There are some errors in "02.qc/baseinfo.txt", please check !!'
		print '\033[0m'
	#~ print infolist
	print >> outfile, '\t'.join(infolist[:6])
	print >> outfile, '\t'.join(infolist[6:])
	outfile.close()

def fastqc(fq1, fq2):
	if not os.path.exists('02.qc/fastqc'):
		os.mkdir('02.qc/fastqc')
	cmd = FASTQC + ' -o 02.qc/fastqc --extract -t 16 -f fastq ' + fq1 + ' ' + fq2
	#~ print cmd
	if os.path.exists(FASTQC):
		os.system(cmd)
	else:
		print '-- Cannot find fastqc in path "/hellogene/scgene01/bio/software/FastQC/fastqc"'
		exit(1)

def getQ20(fq1, fq2):
	if not os.path.exists('02.qc'):
		os.mkdir('02.qc')
	cmd = ITOOLS + ' Fqtools stat -MinBaseQ ! -CPU 4 -InFq ' + fq1 + ' -InFq ' + fq2 + ' -OutStat 02.qc/baseinfo.txt'
	#~ print cmd
	if os.path.exists(ITOOLS):
		os.system(cmd)
	else:
		print '-- Cannot find iTools in path "/hellogene/scgene01/bio/bin/bwa"'
		exit(1)

def statRepli(fqfile1, fqfile2):
	START = 30
	END = 60
	LINE = 4*600000
	fq1 = gzip.open(fqfile1, 'rt')
	fq2 = gzip.open(fqfile2, 'rt')
	outfile = open('02.qc/Repli.txt', 'w')
	seledic = {}
	for i in xrange(1,LINE):
		if (i+2) % 4 == 0:
			read1 = fq1.readline()
			read2 = fq2.readline()
			seq1 = read1[START: END]
			seq2 = read2[START: END]
			seq = "%s%s" % (seq1, seq2)
			#~ seq = seq1 + seq2
			#~ seledic.setdefault(seq, []).append(1)
			if seq in seledic:
				seledic[seq] += 1
			else:
				seledic[seq] = 1
		else:
			fq1.readline()
			fq2.readline()
	rep = 0
	for i in seledic.values():
		if i!= 1:
			rep += (i-1)
	reprate = round(float(rep)*4*100/LINE, 2)
	print >> outfile, 'Repetition Rate: %s%%' %reprate
	outfile.close()

def remap(ref, fq1, fq2):
	if not os.path.exists(ALIGNPATH):
		os.mkdir(ALIGNPATH)
	shell = open(ALIGNPATH + '/r.mp.sh', 'w')
	if not os.path.exists(ref+'.sa'):
		if os.path.exists(BWA):
			index = '/hellogene/scgene01/bio/bin/bwa index '+ref
			os.system(index)
		else:
			print '-- Cannot find bwa in path "/hellogene/scgene01/bio/bin/bwa"'
			exit(1)

	if os.path.exists(BWA) and os.path.exists(SAMTOOLS):
		print >> shell, '#$ -S /bin/sh'
		print >> shell, '%s mem -t 14 %s %s %s | %s view -SbF 0x804 - | %s sort -@ 8 - %s/genome.map' %(BWA, ref, fq1, fq2, SAMTOOLS, SAMTOOLS, ALIGNPATH)
		print >> shell, '%s stats %s/genome.map.bam | grep ^COV | cut -f 2- > %s/map.cov' %(SAMTOOLS, ALIGNPATH, ALIGNPATH)
		print >> shell, '%s depth %s/genome.map.bam > %s/map.dp' %(SAMTOOLS, ALIGNPATH, ALIGNPATH)
		print >> shell, '%s index %s/genome.map.bam' %(SAMTOOLS, ALIGNPATH)
		shell.close()
		#~ qsub = 'qsub -cwd -l vf=4g 02.qc/align/r.mp.sh'
		cmd = 'sh '+ALIGNPATH+'/r.mp.sh'
		os.system(cmd)
	else:
		print '-- Cannot find bwa in path "/hellogene/scgene01/bio/bin/bwa"'
		print '-- OR ...'
		print '-- Cannot find samtools in path "/hellogene/scgene01/bio/bin/samtools"'
		exit(1)

def average(lst):
	num = len(lst)
	summary = sum(lst)
	aver = round(summary / num, 2)
	return aver

def averDepth(depfile):
	outname = os.path.dirname(depfile)+'/averDepth.info'
	outfile = open(outname, 'w')
	dep ={}
	for i in open(depfile, 'r'):
		eachlst = re.findall(r'\w+', i)
		seqid = eachlst[0]
		seqdep = int(eachlst[-1])
		dep.setdefault(seqid, []).append(seqdep)

	newdep ={}
	for k, v in dep.items():
		averdep = average(v)
		newdep.setdefault(k, averdep)
	sortdep = sorted(newdep.iteritems(), key = lambda d:d[1], reverse=True)
	print >> outfile, '%s\t%s' %('Seqid', 'Average_depth')
	for j in sortdep:
		print >> outfile, '%s\t%s' %(j[0], j[1])
	outfile.close()

def statBam(bam, fqfile1):
	outname = os.path.dirname(bam)+'/statBam.info'
	outfile = open(outname, 'w')
	samfile = pysam.AlignmentFile(bam, 'rb')
	align = samfile.count(until_eof=True)

	fq1 = gzip.open(fqfile1, 'rt')
	fq1num = 0
	while True:
		line = fq1.readline()
		if len(line) == 0:
			break
		fq1num += 1
	total = fq1num/2
	#~ total = (fq1num+fq2num)/4 = (fq1num)*2/4 = fq1num/2
	percent = round(float(align)*100/int(total), 2)
	print >> outfile, 'Total Reads: %s' %(total)
	print >> outfile, 'Align Reads: %s' %(align)
	print >> outfile, 'Percent: %s%%' %(percent)
	outfile.close()

def info2xls2(bamFile, averDepFile):
	outname = os.path.dirname(bamFile)+'/Aligninfo.xls'
	outfile = open(outname, 'w')
	print >> outfile, '#%s \t%s \t%s \t%s' %('Total_Reads', 'Align_Reads', 'Align_Percent', 'Average_Depth')
	info = []
	baminfo = open(bamFile, 'r')
	averdep = open(averDepFile, 'r')
	for i in baminfo:
		num = i.strip().split()[-1]
		info.append(num)
	for i in averdep:
		if i.startswith('Seqid'):
			continue
		num = i.strip().split()[-1]
		info.append(num)
	print >> outfile, '\t'.join(info)
	averdep.close()
	baminfo.close()
	outfile.close()

def assReads(fq1, fq2):
	if os.path.exists(SPADES) and os.path.exists(PYTHON):
		shell = open(ASSPATH+'/spade.sh', 'w')
		print >> shell, '#$ -S /bin/sh'
		print >> shell, '%s %s -k 79,97 --only-assembler -1 %s -2 %s -t 15 -m 50 -o %s/spadeAss' %(
				PYTHON, SPADES, fq1, fq2, ASSPATH)
		shell.close()
		qsub = 'qsub -cwd -l vf=50g %s/spade.sh' %(ASSPATH)
		os.system(qsub)

def step2(Args):
	USAGE = "step2 [options] <*fq.gz>"
	parser = argparse.ArgumentParser(usage = USAGE.lower())
	parser.add_argument('-r', dest='reference',
					help="The closed reference of target species, which contain multifasta generally")

	parser.add_argument('-c', dest='closed',
					help="")
	parser.add_argument(dest='fq1', help='R1.fq.gz')
	parser.add_argument(dest='fq2', help='R2.fq.gz')
	if not Args:
		print '\033[1;32;31m',
		parser.print_help()
		print '\033[0m'
		exit(1)
	args = parser.parse_args(args = Args)
	ref = args.reference
	fq1 = os.path.abspath(args.fq1)
	fq2 = os.path.abspath(args.fq2)
	if ref:
		refPATH = os.path.abspath(ref)
		print '\033[1;32;31m',
		print '-- Reads remap to reference',
		print '\033[0m'
		remap(refPATH, fq1, fq2)

		print '\033[1;32;31m',
		print '-- Calculate average depth',
		print '\033[0m'
		averDepth(ALIGNPATH + '/map.dp')

		print '\033[1;32;31m',
		print '-- Calculate align Rate',
		print '\033[0m'
		statBam(ALIGNPATH+'/genome.map.bam', fq1)

		print '\033[1;32;31m',
		print '-- Dispose averdep and align info to xls',
		print '\033[0m'
		info2xls2(ALIGNPATH+'/statBam.info', ALIGNPATH+'/averDepth.info')

	else:
		if not os.path.exists(ASSPATH):
			os.mkdir(ASSPATH)
		print '\033[1;32;31m',
		print '-- Assembly reads by SPAdes',
		print '\033[0m'
		assReads(fq1, fq2)

def getScf(Args):
	USAGE = 'Usage: getscf <scaffolds.fasta>\n <scaffolds.fasta> is the result of SPAdes'
	if not Args:
		print '\033[1;32;31m',
		print USAGE
		print '\033[0m'
		exit(1)
	else:
		ctg = Args[0]
	record = SeqIO.parse(ctg, 'fasta')
	lendic = {}
	seqdic = {}
	count = 0
	outfile = open('First_30_scaffolds.fasta', 'w')
	for i in record:
		if count >= 30:
			break
		print >> outfile, '>%s\n%s' %(i.id, i.seq)
		count += 1
	outfile.close()

#~ ************************************************************
USAGE = '''\
Program: readsQC.py is used to QC raw data (fastq file)
Usage: readsQC.py <command> [option]

Command:
	step1	Doing preliminary QC
	step2	Doing advanced QC
	getscf	Pick 20 lengest scaffolds from the result of SPAdes
Author: Luria
Date: 2016.8.10
'''
#~ ************************************************************

if __name__ == '__main__':
	if len(sys.argv) == 1 :
		print '\033[1;32;31m'+USAGE,
		print '\033[0m'
		exit(1)
	if sys.argv[1] == 'step1':
		if len(sys.argv) == 2:
			print '-- Usage: python %s step1 <*fq.gz>' %(sys.argv[0])
			exit(1)
		else:
			getQ20(sys.argv[2], sys.argv[3])
			fastqc(sys.argv[2], sys.argv[3])
			statRepli(sys.argv[2], sys.argv[3])
			info2xls1('02.qc/baseinfo.txt', '02.qc/Repli.txt')

	elif sys.argv[1] == 'step2':
		step2(sys.argv[2:])

	elif sys.argv[1] == 'getscf':
		'''Because assembly by SPAdes，
			it is better to used scaffolds instead of contigs'''
		getScf(sys.argv[2:])

	else:
		print '\033[1;32;31m'+USAGE
		print '\033[0m'
		exit(1)

*****************************************************************************

2.如果任务失败
重新改变03.ass里的-t 和-m
#$ -S /bin/sh
/hellogene/scgene01/opt/bin/python /hellogene/scgene01/bio/pip/ass/SPAdes-3.5.0-Linux/bin/spades.py -k 79,97 --only-assembler -1 /hellogene/scgene03/prj/SCG-80507-250/SBXZ/01.fq/s80510SBXZ_DSW63404-V_1.clean.fq.gz -2 /hellogene/scgene03/prj/SCG-80507-250/SBXZ/01.fq/s80510SBXZ_DSW63404-V_2.clean.fq.gz -t 12 -m 50 -o 03.ass/spadeAss

查看新产生的03.ass中的warnings.log
查看scaffolds.fasta文件，取前10条序列

3.取前10条序列
先取前10条ID，去除ID前的'>'，保存为list
grep ">" scaffolds.fasta|head -n 10|awk -F '>' '{print $2}' > id.list
安装seqtk
sudo apt install seqtk
使用seqtoolkit按照序列无'>'的ID选取原文件的序列并保存为fasta文件
seqtk subseq scaffolds.fasta id.list > ten.fa

4.在blastn里blast找到线粒体mitochondrial DNA的那条序列按照上面方法找出该序列单独成fa文件one.fa
repeat-match one.fa
结果
Genome Length = 16810   Used 21240 internal nodes
Long Exact Matches:
   Start1     Start2    Length
        1      16714        97

在one.fa里复制一段开始或者末尾的序列查找重复片段，因为线粒体是环形，应该去掉一个重复才能闭合成环


5.排序：将这条线粒体blastn，根据正向sbjct的第1位确定reque序列第1位，如果sbjct位反向，reque序列需先取反向互补后再blastn，根据sbjct确定reque第1位
根据颜色图左边整齐则sbjct为正向，右边整齐则sbjet位反向

反向互补,输出原序列和反向序列
python V1/long_seq2.py in.fa out.fa
去掉输出结果中原来的序列
python V1/long_seq2.py in.fa out.fa|tail -n 2 >rv.fa
*****************************************************************
long_seq2.py文件
from Bio import SeqIO
from Bio.Seq import Seq
import sys

scgene, faf, outf=sys.argv
file=open(faf,'r')
out=open(outf,'w')
for i in SeqIO.parse(file,'fasta'):
	out.write('>%s\n%s\n' % (i.id,i.seq))
	a=Seq(str(i.seq))
	c=a.reverse_complement()
	out.write('>%s\n%s\n' % (i.id,c))
out.close()
****************************************************************
用反向序列再在NCBI里blast比对

6.将第340位的碱基之后的序列移到第1位
vim中3340+right移动到位置
d$将剪切当前位置到行尾的内容
d^将剪切当前位置到行首的内容
d的剪切必须用p粘贴
home键回到第一位，然后p键粘贴

7.对于补了5次都无法补完成的gap需要建库然后在原始scaffold中直接找到
从NNN区域前后各多取2000个碱基出来保存为gap.fa
blast建库`String, `nucl', `prot''
#python find.py seqdump.txt sequence.gff3 > seq.txt
makeblastdb -in scaffolds.fa -dbtype nucl
blastn
blastn -query gap.fa -db scaffolds.fa -outfmt 6 > out.blastn
less out.blastn

makeblastdb -in scaffolds.fa -dbtype prot
blastp
blastp -query gap.fa -db scaffolds.fa -outfmt 6 > out.blastp
less out.blastp

KQ      NODE_7_length_120878_cov_148.579_ID_13  100.00  2000    0       0       1       2000    120842  118843  0.0     3694
KQ      NODE_21_length_1862_cov_1156.19_ID_41   100.00  1501    0       0       2064    3564    362     1862    0.0     2772
KQ      NODE_21_length_1862_cov_1156.19_ID_41   100.00  342     0       0       3722    4063    495     836     0.0     632
序列	名	scaffold中NODE序数，长度，cov和id	相似度	序列长度	snp数量 gap数量	gap.fa开始gap.fa结束scf.fa对应开始scf.fa对应结束	e-velu  score

gap.fa中1:2000都是碱基对应scaffols.fasta中120842:118843是反向序列
由于2000之后是NNN，所以118843之后即是1，最后需要将（118843-1）：1取反向，并多取一部分，确定多取的部分在KQ.fa中NN的前部可以找到才能确定是对的
所以需要取scaffolds.fasta中的NODE_7_length_120878_cov_148.579_ID_13序列，将NODE_7_length_120878_cov_148.579_ID_13保存为list文件，用seqtk subseq scaffolds.fasta list >one.fa
8.取one.fa文件中1:120000个碱基，保存为新文件(before)bf.fa,取反向互补序列使用python long_step2.py bf.fa rvbf.fa
在得到的反向序列rvbf.fa的前端的序列在KQ.fa中可以在NNN的前部找到，然后将没有的部分复制到KQ.fa中NNN的前端

在第二行中可知2000:2064是NNN，所以没有匹配到，2064:3564是匹配的碱基，由于NNN只是代表gap不代表数量，所以对应的362前面的361个碱基需要复制到ＫＱ.fa中NNN的后端，取scaffolds.fa中NODE_21_length_1862_cov_1156.19_ID_41的1:361个碱基，由于2064:3564和362:1862都是正向，所以直接复制给KQ.fa的NNN的后端

此时NNN还不能删除，需要再用bowtie KQ.fa ../*1*.tar.gz ../*2*.fq.gz比对
然后用python spade.py long.srt.bam KQ.fa ../*1*.tar.gz ../*2*.fq.gz 0 1000(1,2文件需要具体修改，0到1000需要根据具体NNN所以位置前后500的范围）
再将scaffolds_long.fasta 与KQ.fa文件搜索补gap，一般需要2次

当只剩下尾部NNN时，需要将首都10kb放在尾部N的后端进行比对并补gap
如果尾部没有NNN也要先在尾部加多个NNN然后将首部的10kb放在尾端比对并补gap

=====================================================================

十五、线粒体注释

1. 以某处作为工作目录
mkdir -p anote/mf anote/sub
# mkdir -p 可以递归建文件
2. 将组装好的 fasta 文件 cp 到 mf 中
3. 进入 mitofish 官网(http://mitofish.aori.u-tokyo.ac.jp/),在网页右侧栏找到
Annotate Mitogenome,上传序列,待网站预测完后下载注释结果到 mf 文件夹下,
并解压
4. 修改一下 step1.sh 中的内容
cd mf
cp ~/bio/MTG/step1.sh .
vim step1.sh
python ~/bio/MTG/mitofish2tbl_2.py C405565.txt C405565_genes.fa XX > t.tbl;
sh ~/bio/MTG/anal.sh C405565_rotated.fa t.tbl
# 强调:只修改标红处!请勿删除_genes.fa 以及_rotated.fa,最终结果是每处对
应 mf 目录下的相应文件。XX 是物种名,为两个用下划线连接的单词,格式如
下 Carassius_auratus
修改完成后,
sh step1.sh
注意：
xx.fa文件与相似种序列排序一定要注意正确第一位对应第一位，否则再次blastn时相似物种的query和sbjct会错位，这时python MTG/genBankRead.py xxxxxxxx.gb > xxxxxxxx.xls注释这一步会报错。
如果是这个报错：
Traceback (most recent call last):
  File "/home/miaobenben/bio/MTG/geneinfo08_1.py", line 171, in <module>
    ReadGenbank(gbf).getStats()
  File "/home/miaobenben/bio/MTG/geneinfo08_1.py", line 68, in getStats
    element.append(feature.qualifiers['gene'][0][:8])
KeyError: 'gene'
则说明找到的近缘物种的genbank文件里没有gene这一项，需要下载其他的有gene这一项的近缘物种的genbank文件
'''*****************************************************************************	     
				   last edit: 2015.11.02		   
*****************************************************************************'''
from Bio import SeqIO
from Bio.SeqUtils import GC
import sys
import re
from xlwt import Workbook,XFStyle,Borders
import xlwt
(xx,gbf)=sys.argv

TRNA={"Ala":"A","Arg":"R","Asp":"D","Cys":"C","Gln":"Q","Glu":"E","His":"H","Ile":"I","Gly":"G","Asn":"N","Leu":"L",
"Lys":"K","Met":"M","Phe":"F","Pro":"P","Ser":"S","Thr":"T","Trp":"W","Tyr":"Y","Val":"V","Sec":"U"," ":""}
base={'A':'T','a':'T','T':'A','t':'A','G':'C','g':'C','C':'G','c':'G'}

class ReadGenbank:
        gbfile=''
        refile=None
        def __init__(self,gbfile):
                self.gbfile=gbfile
                self.refile=SeqIO.read(self.gbfile,'gb')
        
        def getStats(self):
                element=[]
                onelettercode=[]
                codetype=[]
                size=[]
                GCpercent=[]
                beginend=[]
                fromto=[]
                No_ofAA=[]
                initiation=[]
                termination=[]
                anticode=[]
                geneperiod=[]
                intervening=[]
                intergen=[]
                for feature in self.refile.features:
                        seq=''
                        gc=0
                        beginend=[]
                        figure=re.findall(r'\d+',str(feature.location))
                        symbol=re.findall(r'[\+\-]',str(feature.location))[0]
                        chaintype='H' if symbol=='+' else 'L'
                        
                        for j in range(len(figure)/2):
                                beginend.append((int(figure[j*2])+1,int(figure[j*2+1])))
                                seq+=self.refile.seq[int(figure[j*2]):int(figure[j*2+1])]
                        gc=round(GC(seq),2)/100
                        
                        if feature.type=='gene':
                                geneperiod.append([int(figure[0]),int(figure[1])])
                                
                        if feature.type=='CDS':
                                element.append(feature.qualifiers['gene'][0])
                                onelettercode.append(' ')
                                codetype.append(chaintype)
                                size.append(len(seq))
                                AAnum=len(seq)/3-1 if len(seq)%3==0 else len(seq)/3
                                No_ofAA.append(AAnum)
                                GCpercent.append(gc)
                                fromto.append(beginend)
                                initiation.append(self.extractCodon(seq,symbol)[0])
                                termination.append(self.extractCodon(seq,symbol)[1])
                                anticode.append(' ')

                        if feature.type=='tRNA':
                                element.append(feature.qualifiers['product'][0][:8])
                                aa=re.findall(r'\w+',str(feature.qualifiers['product'][0]))[1]
                                onelettercode.append(TRNA[aa])
                                codetype.append(chaintype)
                                size.append(len(seq))
                                No_ofAA.append(' ')
                                GCpercent.append(gc)
                                fromto.append(beginend)
                                initiation.append(' ')
                                termination.append(' ')
				# 2015.11.02 change feature.qualifiers['note'][0])>>>feature.qualifiers['gene'][0])
				genecode=re.findall(r'\w+',str(feature.qualifiers['product'][0]))
				# 2015.10.23 change anticode.append(anti[-1])>>>anticode.append(genecode[-1])
                                # anti=re.findall(r'\w+',str(feature.qualifiers['note'][0]))
                                anticode.append(genecode[-1])
                                
                        if feature.type=='rRNA':
                                element.append(feature.qualifiers['product'][0])
                                onelettercode.append(' ')
                                codetype.append(chaintype)
                                size.append(len(seq))
                                No_ofAA.append(' ')
                                GCpercent.append(gc)
                                fromto.append(beginend)
                                initiation.append(' ')
                                termination.append(' ')
                                anticode.append(' ')

		# if intervening over the range of int, then throw a warn
                for i in range(1,len(geneperiod)):
			intervening.append(geneperiod[i][0]-geneperiod[i-1][1])
			if geneperiod[i][0]-geneperiod[i-1][1]<32766:
                                intergen.append(str(self.refile.seq[geneperiod[i-1][1]:geneperiod[i][0]]))                                
			else: intergen.append('too long, over the range of int')	
                intervening.append(len(self.refile.seq)-geneperiod[-1][1])
		if len(self.refile.seq)-geneperiod[-1][1]<32766:
	                intergen.append(str(self.refile.seq[geneperiod[-1][1]:]))
		else: intergen.append('too long, over the range of int')

                workbook=xlwt.Workbook()
                sheet1=workbook.add_sheet('Genome Base Content',cell_overwrite_ok=True)
                # 2015.10.23 exchange 'InterveningSequence','IntergenicNucleotides'>>>'IntergenicNucleotides','InterveningSequence'
                title=['Gene/element','OneLetterCode','From','To','Size','GC_Percent','Codetype','No. of AA',
                       'InferredInitiationCoden','InferredTermination','Anti-Coden','IntergenicNucleotides',
                       'InterveningSequence']
                fromcol=[]
                tocol=[]
                t=0
                for i in range(len(element)):
                        for j in range(len(fromto[i])):
                                fromcol.append(fromto[i][j][0])
                                tocol.append(fromto[i][j][1])

                                # 2015.10.22 (i+j)>>>t
                                if j!=0:
                                        element.insert(t,' ')
                                        onelettercode.insert(t,' ')
                                        size.insert(t,' ')
                                        GCpercent.insert(t,' ')
                                        codetype.insert(t,' ')
                                        No_ofAA.insert(t,' ')
                                        initiation.insert(t,' ')
                                        termination.insert(t,' ')
                                        anticode.insert(t,' ')
                                        intervening.insert(t,' ')
                                        intergen.insert(t,' ')                                        
                                t+=1

                for i in range(t):
                        sheet1.write(i+1,0,element[i])
                        sheet1.write(i+1,1,onelettercode[i])
                        sheet1.write(i+1,2,fromcol[i])
                        sheet1.write(i+1,3,tocol[i])
                        sheet1.write(i+1,4,size[i])
                        sheet1.write(i+1,5,GCpercent[i])
                        sheet1.write(i+1,6,codetype[i])
                        sheet1.write(i+1,7,No_ofAA[i])
                        sheet1.write(i+1,8,initiation[i])
                        sheet1.write(i+1,9,termination[i])
                        sheet1.write(i+1,10,anticode[i])
                        sheet1.write(i+1,11,intervening[i])
                        sheet1.write(i+1,12,intergen[i])
                            
                for i in range(len(title)):
                        sheet1.write(0,i,title[i])                        
                workbook.save('Gene_Information.xls')

        def extractCodon(self,seq='',symbol='-'):
                if symbol=='-':
                        sequence=''
                        inverseq=seq[::-1]
                        for i in inverseq:
                                if i in base:
                                        sequence+=base[i]
                                else:
                                        sequence+=i
                        start=str(sequence[:3])
                        end=str(sequence[-3:]) if len(sequence)%3==0 else str(sequence[-(len(sequence)%3):])
                else:
                        start=str(seq[:3])
                        end=str(seq[-3:]) if len(seq)%3==0 else str(seq[-(len(seq)%3):])
                return [start,end]

ReadGenbank(gbf).getStats()

经过修改geneinfo08_1.py文件，命令如下产生Gene_Information.xls
python geneinfo08_1.py NC_037044.gb
(根据查找，一般NC_的序列在source下面有gene一项，此处需要用NC_的gb文件，否则会报错）


sequin是一个窗口程序先安装下面程序
sudo apt install ncbi-tools-x11

5. 请对照近缘物种 genbank 生成的表格,认真检查上步运行生成的 xls 表格,此
处认真检查可省去下游分析重复工作!
近缘物种表格生成步骤:
将xxx.fa 文件放到 NCBI 上, blastn 找到最近的近缘物种,下载下来 genbank
文件,运行
python ~/bio/MTG/genBankRead.py xxxxxxxx.gb > xxxxxxxx.xls
# *.gb 是近缘物种 genbank 文件
# AA.xls 中 AA 是这个近缘物种的 Accession number
如果各近缘物种相差较大需要手动注释,再修改 t.tbl 文件

6.对比产生的两个表格中No. of AA和Amino Acids (aa)是否相同
如果不相同则需要将xxx.fa上传到orf finder网站查看
from和to的序列范围根据表格中的from和to确定，然后根据相似物种的genbank中的tranlate table=2则使用的是第2套密码子，则genetic code选择第2项的vertebarate mitochondrial
orf start codon to use选择"ATG" and alternative initiation codons
然后submit提交

在新的orf页面，一般选择第1条包含序列最好的结果，此处一般选择nr而不同prot然后直接blast，勾选方框可在新的页面blast结果

在下方的相似序列中，如果query和sbjct前段和最后段对应则表示没有问题，如果不对应需要先判断是否正向序列，然后根据正反序列和氨基酸代码调整query的序列，注意1个氨基酸对应3个核苷酸，这里调整是query前或后，增或减都是直接调整from和to即可


=================================================================================
BioSoftWare

ncbi-tool(asn2all)				awk
sra-toolkit(fastq-dump)
r-base
scala



1install					MECAT-master
2python						MeV_4_8_1
3sh						MISA
4portable					mito_report
5jar						mothur
6perl						mrbayes-3.1.2
7web						mrbayes-3.2.6					MTG
amos-3.1.0					MUMmer3.23
APBS-1.5-linux64				ncbi-blast-2.2.31+
arachne-46233					ncbi-blast-2.7.1+
art_src_ChocolateCherryCake_Linux		networkx-1.10rc2
auto_barcode-master				NGSQCToolkit_v2.3.3
bamtools					NPDtools-2.1.0-Linux
BarcodeSplitter-master				numpy-1.9.2
bcftools-1.2					OGDraw
BEAST						perl-5.26.0
bedtools2					perlprimer-1.1.21
bfc						perl-tk-master
bio						phylip-3.695.zip
BioList						phylobayes4.1b
bioperl-1.5.2_102				picard-tools-1.119
biopython-1.63					plann-1.1
biopython-1.65					Primer
blasr						primer3-2.2.0-alpha
blast						primer3-2.4.0
bowtie-1.1.2					primer3-py
bowtie2-2.2.5					primer_design
breakdancer-1.1.2				psycopg2-2.6.1
broadinstitute-picard-c8c9f58			pygraphviz-1.3rc2
bwa-0.7.17					pysam-0.7.8
CCT						pysam-0.8.3
Circleator-1.0.0rc4				Python-2.7.10
clustalw-2.1					quast-4.6.3
CMA						R
codonW						R-3.4.0
data						RasMol-2.7.5.2
DEXTRACTOR					rdflib
EMBOSS-6.6.0					rnaQUAST-1.5.1
errmsg						rpy2-2.3.9
ez_setup-0.9					RSEM-1.3.0
fa2txt						RSeQC-2.6.2
FALCON						RVboost_0.1
FastQC						samtools-0.1.19
fastq-tools					samtools-1.2
FastUnifrac					samtools-1.8
fastx_toolkit					seqtk
FLASH-1.2.11					seqtk.git
forTrinity					sequin.linux
fqtools						setuptools-18.0.1
fqtrim-0.9.4					share
FragGeneScan1.20				SOAPdenovo
gatk						software
GATK						SPAdes-3.12.0-Linux
gff3-0.2.0					sratoolkit.2.5.2-centos_linux64
glibc-2.15					standard-RAxML-8.2.11
gwyddion-2.50					SVDetect_r0.8b
hdf5-1.8.16					szip-2.1
hdf5-1.8.16-linux-centos6-x86_64-gcc447-shared	tbl2asn
hdf5-1.8.19					testData
hdf5-1.8.4.snap19-1.el5.x86_64.rpm		treegrahp
htsbox						TreeView-1.1.6r4-bin
htsbox-lite					trinityrnaseq-Trinity-v2.5.1
igrec-3.1.1-Linux				Trinotate-2.0.2
IGV_2.3.55					tRNAscan-SE-1.3.1
images						V1
jellyfish-2.2.5					V2
jmodeltest-2.1.10				VarScan.v2.3.9.jar
jmol-14.29.16					vsearch-1.11.1
jpeg-6b						web.py-0.37
kalign-1.04					wgsim
kmer_project					xerces-c-3.1.1
Lighter						xlrd-0.9.4
linux.gtk.x86_64				xlutils-1.7.1
list.txt					xlwt-0.7.5
mafft-7.245-with-extensions			xlwt-future-0.8.0
maker						y-tools-3.1.1
matplotlib-1.4.3				zlib-1.2.8
mauve_snapshot_2015-02-13


***********************************************************************************************************************************************************
软件使用

1.测序——生成色谱图(峰图文件）（SCF，ABI和预先处理的ESD格式）——得出相应的碱基和质量数（碱基测序质量值Q和碱基出错的概率Pe）——输出fasta格式或者XBAP，PHD，SCF

2.文件格式
Phd文件，用于组装后consed查看编辑；fasta文件为核算序列文件；fasta格式的质量文件

3.Phd2Fasta格式转换
是phred软件包的一部分，将phred产生的phd文件转换成fasta格式的核酸和质量文件，便于crossmatch和phrap程序编辑。

4.载体屏蔽Crossmatch
用于比对两套DNA序列，可以用来找出序列中的载体序列，产生屏蔽了载体的序列，也可用于cDNA和cosmid的比对，相比blastn速度较慢但敏感度高（因允许gap存在）

5.Phrap
华盛顿大学分子生物技术学院的软件包，主要用于shotgun序列的组装。允许使用全长的序列，不仅仅是高质量部分；使用质量信息进行组装提高组装的准确度；能够处理比较大的数据集。

6.Cap3
用于核酸序列拼接的软件，应用正反向信息更正拼接错误，链接contigs；在序列拼接中应用reads的质量信息；自动去reads5'端和3'端低质量区；产生consed程序可读的ace格式拼接结果文件；能用于staden软件包中的cap4软件。

7.Consed
非常强大的图形化finish软件，进行组装的各项统计并绘图，对结果进行比对分析，实现对组装结果进行拆分，重组等功能。

8.primer3
做PCR印务设计，能够比较严格的控制印务发夹结构和二聚体，适合作大规模的PCR引物设计。

9.序列比对
9.1全局比对
Clustalw
核酸、蛋白质的全局比对，为构建分子进化树分析提供基础，

10.cpAno/cleanCP2.py
叶绿体注释结果文件*.fas和*.gff转成t.tbl和error.log文件
python cpAno/cleanCP2.py *.fas *.gff 1>t.tbl 2>error.log

11.Sequin
asn文件放到Sequin里边去，再选择error，之后export出来，之后最好的确定ORF的方法是直接在Sequin中Annotate菜单 -> ORF Finder

12.readsQC.py
python readsQC.py step1 *.gz
将所有原始fq文件的gz文件处理生成.qc文件夹，查看生成的Basicinfo.xls中GC,Q20,Q30和RepRate的值，如果Q30的之少于80%，则需要去除质量不好的reads。

python readsQC.py step2 01.fq/R*
组装过程

13.Trimmomatic
sudo java -jar trimmomatic-0.36.jar PE -threads 12 -phred64 R1.fq R2.fq R1.fa R1_r.fa R2.fa R2_r.fa AVGQUAL:1
用Trimmomatic取出低质量的reads，剪切reads，完成后把R1.fa和R2.fa 重新运行qc一步readsQC.py ，如Q30的值没问题即可继续．

Trimmomatic的用法：
java -jar /sam/trimmomatic/trimmomatic-0.32.jar PE -threads 12 -phred64 -trimlog logfile input_forward.fq input_reverse.fq output_forward_paired.fq output_forward_unpaired.fq output_reverse_paired.fq output_reverse_unpaired.fq ILLUMINACLIP:1.adapter.list:2:30:10 LEADING:3 TRAILING:3 SLIDINGWINDOW:4:15 MINLEN:36
同一个指令下的不同参数可以用':'来界定
phred33    设置碱基的质量格式,如果不设置，默认的是-phred64
trimlog file就是产生日志，包括如下部分内容：read的名字，留下来的序列的长度，第一个碱基的起始位置，从开始trimmed的长度，最后的一个碱基位于初始read的位置，最后trimmed的数量，其他步骤产生的日志
LEADING:3   切除首端碱基质量小于3的碱基或者N
TRAILING:3   切除末端碱基质量小于3的碱基或者
ILLUMINACLIP: 1.adapter.lis:2:30:10 1.adapter.list为adapter文件，允许的最大mismatch 数，palindrome模式下匹配碱基数阈值：simple模式下的匹配碱基数阈值
SLIDINGWINDOW:4:15  Windows的size是4个碱基，其平均碱基质量小于15，则切除
MINLEN:36  最低reads长度为36
CROP:  保留的reads长度
HEADCROP: 在reads的首端切除指定的长度
下载地址：http://www.usadellab.org/cms/?page=trimmomatic （注意里面download那里，选择source和binary）

14.canu
大的全基因组的组装（有三代和二代的数据），先用canu组装（纯三代reads组装），然后再用pilon（https://github.com/broadinstitute/pilon/wiki）去补gap和修正snp，indel。

15.普通鱼类线粒体基因组QC流程
一般鱼类(脊椎动物)有:13 个编码基因,22 个 tRNA,2 个 rRNA,1 个 D-loop
区。

16.Annotation 中 figure 图 QC 点
1、 确定 figure 图是否有 pdf,png,tif 格式;
2、 确定 figure 图中是否有箭头标注方向;
3、 物种拉丁名是否是斜粗体,且在内圈里;
4、 D-LOOP 区是否标出,如果注释结果是 misc-featrue,圈图中也要标出。

17.pilon-1.22.jar
java -Xmx100G -jar /hellogene/scgene02/RD/RiceHybridDenovo/bin/pilon-1.22.jar --genome ../index/rice.unitigs.fasta --frags ../mem/unitigs.sort.bam --output unitigs
用pilon补SNP和gap

18.kmer的值
由于进行的是低深度的测序，所以一般情况下kmer不适宜设计过大，一般不建议超过55。具体kmer设多大，可以根据jellyfish跑出来的kmer分布图确认基因组大小后再设kmer的值，也可通过参考近缘物种的基因组大小进行设置。

19.assemble.sh组装程序：
/hellogene/scgene03/RD/RADPlus/1assemble/assemble.sh
内容：
#$ -S /bin/sh
/hellogene/scgene03/RD/RADPlus/bin/SOAPdenovo2-bin-LINUX-generic-r240/SOAPdenovo-63mer all -s assemble.conf -o gyGenome -K 43 -p 32

20.spade.sh
#$ -S /bin/sh
/hellogene/scgene01/opt/bin/python /hellogene/scgene01/bio/pip/ass/SPAdes-3.5.0-Linux/bin/spades.py -k 79,97 --only-assembler -1 /hellogene/scgene02/prj/SCG-61021-169AB/HB/01.fq/1.fq -2 /hellogene/scgene02/prj/SCG-61021-169AB/HB/01.fq/2.fq -t 15 -m 30 -o /hellogene/scgene02/prj/SCG-61021-169AB/HB/03.ass/spade_text/spadeAss

21.r.mp.sh
/hellogene/scgene01/bio/bin/bwa mem -t 14 HB.fa ../../01.fq/51112SYH_L6_I376.R1.clean.fastq.gz ../../01.fq/51112SYH_L6_I376.R2.clean.fastq.gz | /hellogene/scgene01/bio/bin/samtools view -SbF 0x804 - | /hellogene/scgene01/bio/bin/samtools sort -@ 8 -  genome.map
/hellogene/scgene01/bio/bin/samtools stats genome.map.bam | grep ^COV | cut -f 2- > map.cov
/hellogene/scgene01/bio/bin/samtools depth genome.map.bam > map.dp
/hellogene/scgene01/bio/bin/samtools index genome.map.bam 

22.jellyfish统计基因组大小
/hellogene/scgene01/bio/bin/jellyfish count -C *.fq -m 21 -s 100M -t 24 -o kmer_count

#-C后面可接几个fa或者fq文件
大型机上面用which jellyfish 的地址是错误的，是在/hellogene/scgene01/bio/bin/jellyfish 而不是/hellogene/scgene01/opt/bin/jellyfish

然后/hellogene/scgene01/bio/bin/jellyfish histo kmer_count > histo.txt

再打开histo.txt文件，找出第二列出现第一个最低值后出现的最大值，给出第一列的值x。

把总的碱基的数量除以x得到基因组大小。统计输入的fq文件的行数y，然后计算y/4*2*150/x 出来的值就是预计的基因组大小（如太小则可能是fq的q质量过低，应先去除低质量的reads）

23.CASAVA
Illumina高通量测序结果最初以原始图像数据文件存在，经CASAVA软件进行碱基识别（Base Calling）后转化为原始测序序列（Raw Reads）

24.NCBI 提交
将已完成的基因组序列和基因注释，按照NCBI要求制作成可以直接提交NCBI的序列文件，直接将 submit2ncbi目录中的SQN文件以附件的形式发送给NCBI官方邮箱（gb-sub@ncbi.nlm.nih.gov），即完成提交。
NCBI提交相关文件请查看“Submit 2 NCBI”文件夹。、
生成的NCBI提交文件默认是提交并通过审核后就马上公开该序列，这个过程一般需要1到2个月的时间，如果希望在确定的时间公布该序列，请联系我们修改发布时间后，再做提交。

25.Codon Usage分析
线粒体密码子的使用具有偏好性（Codon Usage Bias），密码子偏好性在一定程度上会影响基因的表达和反映物种的进化关系[28]。利用SCGene整合的Codon Usage分析流程，对Hariotina reticulata线粒体基因组64种密码子的使用进行统计

26.sequencing
质量检测

27.chloroplast & mitochondrion assemble（CMA）
序列捕获,基因组组装与验证

28.blastn & blastx (NCBI)
基因组注释

MiTFi(University of Leipzig)
基因组注释

tRNAscan-SE(UCSC)
基因组注释

OGDRAW(Max Planck Institute)
基因组注释

29.tbl2asn & sequin(NCBI)
基因序列提交

30.Mega6 Mega7
进化分析

31.MUSCLE
进化分析

32.jModelTest
进化分析

33.RAxML
进化分析

34.Clustal Omega
基因差异比较分析

35.fa2plyp.py
python fa2plyp.py dna.fa dna.phy
将fasta格式转化为phylip格式

36.多序列比对软件对比
如果选择用RAxML来建ML树，则需用MEGA进行比对后再转化为phylip格式（*.phy），RAxML只能识别这种比对格式文件。

MEGA比对掐头去尾后输出FASTA格式（*.fas）文件（同时也要生成*.nex格式文件用于后期检测饱和度）

将fasta格式转化为phylip格式
python fa2plyp.py dna.fa dna.phy

ClustalX和集群上cds.sh中muscle只能比对但不能掐头去尾，可以使用ClustalX转化为*.phy格式文件。

打开ClustalX，File-> Load Seqences将MEGA产生的文件载入-> 菜单栏Alignment-> Output Format Options-> Output Files下改选PHYLIP format-> 菜单栏Alignment ->Do Complete Alignment -> OK

37.建ML树
可选软件有RAxML，Treefinder, Phyml(据说是最快的进化ML树的软件)，Garli。综合软件有PHYLIP，MEGA，PAUP。（虽然MEGA也可以做ML，但模型太少，不建议。）
在集群上我们使用的RAxML。

37.1.1核苷酸树如下：
#$ -S /bin/sh
evolution/raxml -s nucl.phy -m GTRGAMMAI -f a -N 500 -T 32 -x 855500 -p 23444 -n dna_b1000
HKY+I+G=HKYGAMMAI
# -m 后接4.2算出的最佳模型
# -t 后接cpu数，-f接-f a， 选择算法，a 代表同时做快速 bootstrap 分析和搜索最佳 似然树【默认是 -f d】 ，-p 一个用于简约推断的随机数，-N bootstrap 重复次数，一般以 1000 为宜，-T 线程数量，不超过 32， 不超过计算机的总线程数量，-n 输出文件名，-s phylip 或 fasta 格式 alignment 序列
保存后，用qsub -cwd -l vf=10G ml.dna.sh抛到集群去运算
如果有报错，错误原因会写在“*.o*”的文件中。

37.1.2氨基酸树则如下：
#$ -S /bin/sh
evolution/raxml -s aa.phy -m PROTGAMMAILGF -f a -N 500 -T 32 -x 855500 -p 23444 -n dna_b1000
#LG 换成MTMAM（最佳模型为MtMam+I+G+F)

37.2vim建一个名为ch.sh的脚本，脚本里写
#$ -S /bin/sh
cp RAxML_bipartitionsBranchLabels.dna_b1000  tree.nwk
awk '{print $1,$2,$3}' taxonomy.info.xls | while read a b c ;do echo "sed -i \"s/${b}_$c/'${b} $c ($a)'/\" tree.nwk ";done > s.sh
Taxonomy.info.xls是t.info时要注意物种名是否有更改，如是tox.xls则要注意是否需要NC号后面的.1
# 执行完生成一个原始的tree.nwk和s.sh文件
sh s.sh 
# 运行生成的s.sh 修改成最后的tree.nwk文件

37.3
把tree.nwk文件导入mega中进行树的可视化修改。
有关Bootstrap和Jackknifing
bootstrap:以每一列为单位，随机排放各列(行的长度始终相等)，产生不同的多序列
jackknifing:随机取一半，再随机取一半的一半，再随机取一半的一半的一半

38.cutadapt去除接头:cutadapt -O 5 -m 50

39.pear序列拼接:pear -x 0.1

40.prinseq质量剪切:prinseq -lc_method dust -lc_threshold 40 -min_len 200

41.mothur做rerefaction分析
R做rarefaction curve稀释曲线

42.R做rank-abundance曲线

43.R的vegan package做species accumulation curves物种累计曲线

44.R对物种分类学统计结果作图

45.graphlan，metaphlan2和itol,根据每个样本的分类学比对结果，选出优势种的分类，结合物种丰度信息，以环状图显示

46.pandoc:将markdown文件转为docx等格式的工具，使用python，内有latex
pandoc file.md -o file.docx

47.mafft:比对速度（Muscle>MAFFT>ClustalW>T-Coffee），比对准确性（MAFFT>Muscle>T-Coffee>ClustalW）。因此，推荐使用 MAFFT 软件进行多序列比对。

48.Bioconda是自动化管理生物信息软件的工具，就像 APPStore、360 软件管家一样。配置好之后，安装 seqtk 只需要一条命令 conda install seqtk

49.trinity将kmer片段拼接，默认拼接大小为25，上限是32










*********************************************************************************

十六、基因重排

Circos 作图

把snp放到环形图里面，文件夹SNP是例子，*.snp是不同样品的snp的位置，circos.conf是主程序，不知道什么原因，像基因和snp起始结束的文件第三列数值最低的的不显示，所以要加一行比其他数值要小的行：如下列红色字体，

hs1 0 1537 70.0000 fill_color=yellow

hs1 1538 1603 70.0000 fill_color=blue

hs1 0 1 30.0000

Circos.conf中karyotype=序列的大小

Plots中第一个plot是修改基因的半径，颜色，file是基因起始和终止的文件，也需要 另外加一行hs1 0 1 30.0000

下面的plot是snp文件的，color是线的颜色，填充颜色一般显示不出来，颜色名字前加d是加深，vd是更深，vvd是最深，变浅l同理。extend_bin  = no是必须的，这个是是否把snp连在一起的选项，



基因重拍：

请看基因重排文件夹，已经基本完成

chromosomes                 = /hs[12]$/ 是选择显示的ID（即对应文件的第一列）

===================================================================================

十七、新的codonusage的做法
1.新建一个目录，把bio/MTG/codonusage的文件夹复制到当前文件夹，cp -r ~/bio/MTG/codonusage ./ 
2.把sqn文件复制到codonusage的文件夹内运行sh step1 *.sqn。程序出来结果的是以第一套密码子做的，所以需要更改密码子。进入CU1,根据物种的所使用的密码子的套数，运行python ../bin/change_codon.py *.csv 2 fre.csv (其中*.csv是本来存在的csv文件，2是密码的套数，目前程序只支持第2,4,5,22,23这几套的转换，其他的需到程序中添加，fre.csv是出来的结果，名字必须为fre.csv)。
3.用wps打开fre.csv ,先把Frequency一列降序，再Amino acid一列升序（是为了使长的柱子在下面）。在后面新加入一列Countif：=COUNTIF($A$2:A2,A2)
运行sh step2. 生成frequency.pdf和RSCU.pdf。在photoshop中对图片进行调整，另存为pdf,png,tif三种格式。模板如下图：
5.打开RSCU.csv，格式调整为codonusage文件夹下的codonUsage模板.xls的样子，其中字体全文time new roman.1和2列列宽为12,3到5列列宽为16，除了备注行，其余行高为18.
*********************************************************************************
CodonUsage图的生成
注意：是relative synonymous codon usage (RSCU)
V_3版的作了一些整合，最后的操作如下：
1. 先将V_3版整个文件夹拷贝到一处作为工作目录
2. 将t.info文件（格式为：每行一个，物种id+“\t”+物种名）拷贝到工作目录下，与step1.sh同级目录下。
注意：文件名必须为t.info，该文件中不加目标物种
3. sh step1.sh  
# 执行step1.sh脚本，该脚本的目的：a.在NCBI后台下载下近缘物种CDS; b.此处生成的是以NC号命名的文件，而后期要用的是以物种名命名的文件，因此需通过Underline.py程序转换,此过程会产生一个临时文件，执行完后会自动删掉；c. 为了让所有的CDS区合在一起还能正常表达，需要运行bin目录下Mcds.py将终止密码子（完全和不完全的）去掉，清洗数据，同时完成每个物种中所有CDS区的合并； 
4. 将目标物种CDS文件拷贝到工作目录下，与step1.sh同级目录下
asn2all -i XX.sqn -o XX.cds -f d
# XX.sqn是目标物种的sqn文件
# 运行完生成XX.cds文件，即需要的CDS文件
注意：不是目标物种的基因组fasta文件！
5.python bin/Mcds.py XX.cds > XX.fa
# 完成目标物的清洗和整合，XX.fa为下划线连接的物种名（如：Cephalloscyllium_umbratile.fa）
6.将产生的XX.fa文件拷贝一份到all文件夹中，拷贝一份到CU1中，修改一下CU1中的01.sh（*.fa修改为第5步中生成的XX.fa）生成了一个csv表。
注意：
A.此时产生的*.fa.csv文件是基于第1套密码子表，需运行Python ../bin/change_codon.py *.fa.csv 2 fre.csv (数字为第几套密码子，目前程序只有第2,4,5,22这几套密码子，其他的请自行补充)
把其中Frequency列中为0的行删除，为了让条形图中较长的柱子在下面，需按Frequency(%)列降序排列，再按Amino acid用Counif函数
例如：=COUNTIF($A$2:A2,A2)
运行python get_RSCU.py fre.csv RSCU.csv
再以此在R中画图
Frequency图：
Rscript Script_v3_Frequency.R fre.csv
RSCU图：
Rscript Script_v3_RSCU.R RSCU.csv

# XX.fa.csv是执行01.sh生成的文件
# 生成RSCU.pdf需要在PS中调整一下
7.在CU2中(需要多个物种）
ln -s ../all .
ls all > all.lst
sh 01.sh 
# 生成raw.ma但是raw.ma是按第1套密码子表生成的，因此下步还得转化为第23套。
vim ../bin/Codon.py添加if = usedcode==’’：...
python ../bin/Codon.py raw.ma 23 > cu.ma2
# raw.ma 是需改变的表；23是第23套密码子表
把raw.ma改成cu.ma运行后产生cu.ma2文件，再在Rstudio中运行step2.R，以cu.ma为原始文件产生热图（如字体看不清晰，可打开step2.R中的fontsize中调节字体大小）。

***********************************************************************************
step1.sh文件

mkdir cds
echo 'Downloading CDS ***'
awk '{print $1}' t.info | while read l;do asn2all -r t -A $l -o cds/$l.cds -f d;done
echo 'Downloading OVER ***'
python bin/Underline.py
mkdir all
cat U.info | while read m l;do python bin/Mcds.py cds/$m.cds > all/$l.fa;done
rm U.info
echo 'Remove Temp File OVER ***'
*******************************************************************************
step2.R文件
library(pheatmap)

# draw F.A
ma <- read.table("cu.ma",header=TRUE,row.names=1,check.names=FALSE)
pheatmap(ma,color=colorRampPalette(c("blue","green","red"))(50),cluster_row=FALSE,fontsize=8,fontsize_row=12)
pheatmap(ma,color=colorRampPalette(c("blue","green","red"))(50),fontsize=8,fontsize_row=12)



# draw F.B
mat <- t(ma)
write.table(mat,file="cu.ma.t")
mt <- read.table("cu.ma2",header=TRUE,row.names=1,check.names=FALSE)
mma <- t(mt)
pheatmap(mma,color=colorRampPalette(c("blue","green","red"))(50),cluster_cols=FALSE,cluster_row=T,fontsize=8,fontsize_row=12)



=====================================================================================
Kmer

python如何将给的碱基序列切成长度为4的kmer
kmer.py文件：
# 定义方法
def get_list(seq,n):
    """
    Parameters:
    ---------
        seq: string
        n : integer
    """
    result = []
    for i in range(len(seq)-n+1):
        result.append(seq[i:i+n])
    return result



一、关于Fastq

FASTQ是基于文本的，保存生物序列（通常是核酸序列）和其测序质量信息的标准格式。其序列以及质量信息都是使用一个ASCII字符标示，最初由Sanger开发，目的是将FASTA序列与质量数据放到一起，目前已经成为高通量测序结果的事实标准。

二、Fastq的格式

FASTQ文件中每个序列通常有四行：第一行，序列标识以及相关的描述信息，以‘@’开头；第二行是序列；第三行以‘+’开头，后面是序列标示符、描述信息，或者什么也不加；第四行，是质量信息，和第二行的序列相对应，每一个序列都有一个质量评分，根据评分体系的不同，每个字符的含义表示的数字也不相同。

三、关于Fasta

Fasta格式也称为Pearson格式，是一种基于文本用于表示核苷酸序列或氨基酸序列的格式。在这种格式中碱基对或氨基酸用单个字母来编码，且允许在序列前添加序列名及注释。

四、Fasta格式

Fasta格式首先以大于号“>”开头，接着是序列的标识符；换行后是序列的描述信息。换行后是序列信息，文件每行的字母一般不应超过80个字符。序列中允许存在空格，换行，空行，直到下一个大于号或文件结束，表示该序列的结束。

在clean.fq的fastq文件中取n条序列，1部分为4行内容，1行为信息，2行为碱基序列，3行为+，4行为质量分数
seqtk sample -2 -s100 RK80125WYL1_HK7KVCCXY_L1_1.clean.fq.gz 10 > sub1.fq

$ seqtk sample

Usage:   seqtk sample [-2] [-s seed=11] <in.fq> <frac>|<number>

Options: -s INT       RNG seed [11]
         -2           2-pass mode: twice as slow but with much reduced memory
[]中的是可选参数，<> 中的是必需参数。
[-2] 内存较小的服务器上运行时，设置此参数。
[-s] 随机数的种子。如果是 Pair-end 数据，需要保证 read1 和 read2 的种子一致，才能抽到相同的raeds。默认是 11。
[in.fq] 输入文件

<frac|number>可以输入要抽取的比例或 reads 条数。

三种错
tips:在kmer拼接过程中出错产生的分支
bubbles：在kmer拼接过程种出错产生的环
覆盖不均一

==================================================================
java环境配置
设置环境变量：

#vi /etc/profile

在最后面加入

export JAVA_HOME=/usr/java/jdk1.8.0_73
export PATH=$JAVA_HOME/bin:$PATH
export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar

按:wq保存退出。

4.使配置生效

source /etc/profile
====================================================================
列出当前目录下所有的文件夹
ls -l |awk '/^d/ {print $NF}'

======================================================================
KEGG,KOG,GO

run.sh文件：
#$1表示shell传入的第一个参数，此处为文件夹名字
mkdir "new_"$1

#kegg
cd "new_"$1
#判断每行的数据都不为0则取其id输出保存
awk '{if($2 != 0 && $3 != 0 && $4 != 0 && $5 != 0 &&6 != 0 && $7 != 0)print $1".1"}' ../$1/gene_count_matrix.csv > list
#deAnno,在all.pep.kegg.blastp.1库文件中找出第一列中有list内第一列字符串的行并保存为kegg.blastp,即蛋白质比对的结果
python /hellogene/scgene02/RD/RNASeqRef/P135/04Enrichment/bin/deAnno.py list /hellogene/scgene02/RD/RNASeq/Oryza/all.pep.kegg.blastp.1 > kegg.blastp
#geneAnnoKEGG
python /hellogene/scgene02/RD/RNASeq/bin/geneAnnoKEGG.py kegg.blastp > kegg.blastp.out
#keggGene2Pathway
python /hellogene/scgene02/RD/RNASeq/bin/keggGene2Pathway.py kegg.blastp.out > kegg.pathway
#filtPathwayByOrg
python /hellogene/scgene02/RD/RNASeq/bin/KEGG/filtPathwayByOrg.py /hellogene/scgene02/RD/RNASeq/db/KEGG/listPathway/osa kegg.pathway > kegg.pathway.filt
mkdir kegg
python /hellogene/scgene02/RD/RNASeq/bin/KEGG/Pathway2Plot.py kegg.pathway.filt kegg

#go
#GOAnno
python /hellogene/scgene02/RD/RNASeqRef/P135/04Enrichment/bin/deAnno.py list /hellogene/scgene02/RD/RNASeq/Oryza/all.pep_uniprot.blastp.1 > uniprot.blastp
#GOAnno
python /hellogene/scgene03/prj/SCG-80530-258/03.remap/bin/GOAnno.py uniprot.blastp > go.anno
GODistribution
python /hellogene/scgene03/prj/SCG-80530-258/03.remap/bin/GODistribution.py go.anno > go.distribution
mkdir go
python /hellogene/scgene02/RD/RNASeq/bin/GO/GO2Plot.py go.distribution go

#kog
#判断每行的数据都不为0则取其id输出保存
awk '{if($2 != 0 && $3 != 0 && $4 != 0 && $5 != 0 &&6 != 0 && $7 != 0)print $1}' ../$1/gene_count_matrix.csv > kog.list
#deAnno
python /hellogene/scgene02/RD/RNASeqRef/P135/04Enrichment/bin/deAnno.py kog.list /hellogene/scgene03/prj/SCG-80403-239/DB/kog.result.anno > kog.anno
mkdir kog
python ../kog.py kog.anno kog
cp ../kog2plot.R kog
cd kog
Rscript kog2plot.R
cd ../../

******************************************************************************
deAnno.py文件：
#! /usr/bin/env python
import sys
import re

def main(de, anno):
    de_dic = {}
    for line in open(de):
        line = line.rstrip()
        lines = line.split()
        de_dic[lines[0]] = 0

    for line in open(anno):
        line = line.rstrip()
        lines = line.split()
        if lines[0] in de_dic:
            print line

if len(sys.argv) > 1:
    main(sys.argv[1], sys.argv[2])

all.pep.kegg.blastp.1库文件和kegg.blastp结构相同，包含关系
LOC_Os01g01010  osa:4326813     96.033  731     0       1       1       702     1       731     0.0     1450
LOC_Os01g01019  osa:4326455     100.000 87      0       0       1       87      1       87      8.99e-51        176
LOC_Os01g01030  osa:4326455     100.000 593     0       0       1       593     137     729     0.0     1221


geneAnnoKEGG.py文件：
#! /usr/bin/env python
import sys
import re


def main(blast_hit):
    anno = "/hellogene/scgene02/RD/RNASeq/db/KEGG/ko.list"
    ko2gene = "/hellogene/scgene02/RD/RNASeq/db/KEGG/KO2Gene.join.txt"
    blast_hit_dic = {}
    anno_dic = {}
    for line in open(blast_hit):
        line = line.rstrip()
        lines = line.split()
        isoform_id = lines[0]
        gene_id = lines[0]
        if lines[0].split("_")[-1][0] == "i":
            gene_id = lines[0].split("_")[0]
        ref_id = lines[1]
        hit_score = float(lines[-1])

        if gene_id in blast_hit_dic:
            if blast_hit_dic[gene_id][-1] > hit_score:
                continue
        blast_hit_dic[gene_id] = [isoform_id, ref_id, hit_score]
        anno_dic[ref_id] = []

    ko_anno_dic = {}
    for line in open(anno):
        line = line.rstrip()
        lines = line.split("\t")
        ko_anno_dic[lines[0]] = lines[1]

    for line in open(ko2gene):
        line = line.rstrip()
        lines = line.split()
        if lines[1] in anno_dic:
            anno_dic[lines[1]] = [lines[0], ko_anno_dic[lines[0]]]

    print "#unigene\tkegg\tidentity(%)\tmapLen(aa)\tmismatch(aa)\tgap(aa)\tstart(unigene)\tend(unigene)\tstart(kegg)\tend(kegg)\te-value\tscore\tannotation"
    remove_dup_dic = {}
    for line in open(blast_hit):
        line = line.rstrip()
        lines = line.split()
        isoform_id = lines[0]
        gene_id = lines[0]
        if lines[0].split("_")[-1][0] == "i":
            gene_id = lines[0].split("_")[0]
        ref_id = lines[1]
        if isoform_id == blast_hit_dic[gene_id][0]:
            if gene_id in remove_dup_dic:
                continue
            remove_dup_dic[gene_id] = 0
            if len(anno_dic[ref_id]) == 0:  #undistributed KID
                continue
            lines[0] = gene_id
            lines[1] = "%s(%s)" % (anno_dic[ref_id][0].split(":")[1], lines[1])
            print "%s\t%s" % ("\t".join(lines), anno_dic[ref_id][1])


if len(sys.argv) > 1:
    main(sys.argv[1])
else:
    print >>sys.stderr, "python %s <blast>" % (sys.argv[0])


ko.list文件格式
ko:K00001       E1.1.1.1, adh; alcohol dehydrogenase [EC:1.1.1.1]
ko:K00002       AKR1A1, adh; alcohol dehydrogenase (NADP+) [EC:1.1.1.2]
ko:K00003       E1.1.1.3; homoserine dehydrogenase [EC:1.1.1.3]
KO2Gene.join.txt文件格式
ko:K00001       dme:Dmel_CG3481
ko:K00001       dpo:Dpse_GA17214
ko:K00001       dan:Dana_GF14888
ko:K00001       der:Dere_GG25120

kegg.pathway文件格式
#pathwayID      pathway koNum   geneNum genes
00010   Glycolysis / Gluconeogenesis    32      128     LOC_Os05g44922;LOC_Os04g39420;LOC_Os06g05860;LOC_Os01g53680;LOC_Os01g09570;LOC_Os05g10650;LOC_Os09g24910;LOC_Os08g34050;LOC_Os10g26570;LOC_Os09g20820;LOC_Os03g15950;LOC_Os03g14450;LOC_Os10g08550;LOC_Os06g04510;LOC_Os09g26880;LOC_Os06g22060;LOC_Os08g25720;LOC_Os06g13810;LOC_Os02g48360;LOC_Os09g12650;LOC_Os11g10480;LOC_Os11g10510;LOC_Os01g67860;LOC_Os05g33380;LOC_Os08g02700;LOC_Os01g02880;LOC_Os06g40640;LOC_Os11g07020;LOC_Os06g14510;LOC_Os03g56460;LOC_Os09g29070;LOC_Os08g37380;LOC_Os12g05110;LOC_Os07g08340;LOC_Os11g05110;LOC_Os11g10980;LOC_Os04g58110;LOC_Os03g46910;LOC_Os01g16960;LOC_Os03g20880;LOC_Os10g42100;LOC_Os01g47080;LOC_Os12g08170;LOC_Os06g01630;LOC_Os08g33440;LOC_Os07g22720;LOC_Os06g30460;LOC_Os02g01500;LOC_Os04g40950;LOC_Os02g38920;LOC_Os02g07490;LOC_Os08g03290;LOC_Os06g45590;LOC_Os08g34210;LOC_Os10g11140;LOC_Os03g50480;LOC_Os02g51590;LOC_Os02g32490;LOC_Os04g33190;LOC_Os02g01510;LOC_Os06g01590;LOC_Os03g60370;LOC_Os05g09500;LOC_Os01g52450;LOC_Os05g44760;LOC_Os01g09460;LOC_Os01g53930;LOC_Os07g26540;LOC_Os07g09890;LOC_Os01g71320;LOC_Os05g45590;LOC_Os01g46950;LOC_Os09g15820;LOC_Os02g08030;LOC_Os08g14330;LOC_Os04g51390;LOC_Os04g56290;LOC_Os05g49430;LOC_Os05g36270;LOC_Os01g64660;LOC_Os06g45370;LOC_Os03g16050;LOC_Os03g15050;LOC_Os06g45710;LOC_Os01g58610;LOC_Os05g41640;LOC_Os02g07260;LOC_Os10g07229;LOC_Os11g10520;LOC_Os09g36450;LOC_Os01g05490;LOC_Os01g62420;LOC_Os06g13720;LOC_Os02g50620;LOC_Os04g02900;LOC_Os03g44300;LOC_Os12g42230;LOC_Os08g42410;LOC_Os09g33500;LOC_Os06g15990;LOC_Os04g45720;LOC_Os12g07810;LOC_Os11g08300;LOC_Os02g49720;LOC_Os02g43194;LOC_Os02g43280;LOC_Os04g38540;LOC_Os04g38530;LOC_Os09g02140;LOC_Os10g06720;LOC_Os02g36600;LOC_Os01g06660;LOC_Os05g39320;LOC_Os05g39310;LOC_Os03g18220;LOC_Os07g49250;LOC_Os07g02030;LOC_Os01g22520;LOC_Os05g06460;LOC_Os01g23610;LOC_Os05g06750;LOC_Os07g42924;LOC_Os02g57040;LOC_Os03g08999;LOC_Os03g09020;LOC_Os08g37140;LOC_Os05g40420;LOC_Os01g60190

keggGene2Pathway.py文件
#! /usr/bin/env python
import sys
import re

def main(blast_hit_anno):
    kid2pathway = "/hellogene/scgene02/RD/RNASeq/db/KEGG/KID2Pathway.list"
    pathway_anno = "/hellogene/scgene02/RD/RNASeq/db/KEGG/pathway.list"
    pathway_anno_dic = pathwayInfo(pathway_anno)

    kid_dic = {}
    for line in open(blast_hit_anno):
        line = line.rstrip()
        if re.search("^#", line):
            continue
        lines = line.split()
        gid = lines[0]
        kid = lines[1]
        score = float(lines[11])
        if kid not in kid_dic:
            kid_dic[kid] = ["", 0]
        if score > kid_dic[kid][-1]:
            kid_dic[kid][0] = gid
            kid_dic[kid][1] = score

    kid_join_dic = {}
    for kid in kid_dic:
        pkid = kid.split("(")[0]
        if pkid not in kid_join_dic:
            kid_join_dic[pkid] = []
        kid_join_dic[pkid].append(kid_dic[kid][0])

    pathway_dic = {}
    for line in open(kid2pathway):
        line = line.rstrip()
        lines = line.split()
        map_id = lines[0].split(":")[1][3:]
        kid = lines[1].split(":")[-1]
        #print map_id, kid
        if kid in kid_join_dic:
            if map_id not in pathway_dic:
                pathway_dic[map_id] = {}
            pathway_dic[map_id][kid] = kid_join_dic[kid]

    print "#pathwayID\tpathway\tkoNum\tgeneNum\tgenes"

    for i in sorted(pathway_dic):
        kid_num = 0
        unigene_num = 0
        unigene_dic = {}
        unigenes = []
        for j in pathway_dic[i]:
            kid_num += 1
            for k in pathway_dic[i][j]:
                if k not in unigene_dic:
                    unigene_dic[k] = 0
                    unigenes.append(k)
                    unigene_num += 1
        print "%s\t%s\t%d\t%d\t%s" % (i, pathway_anno_dic[i], kid_num,\
                unigene_num, ";".join(unigenes))


def pathwayInfo(pathway_anno):
    pathway_anno_dic ={}
    for line in open(pathway_anno):
        line = line.rstrip()
        lines = line.split("\t")
        map_id = lines[0].split(":")[1][3:]
        map_anno = lines[1]
        pathway_anno_dic[map_id] = map_anno

    return pathway_anno_dic


if len(sys.argv) > 1:
    main(sys.argv[1])
else:
    print >>sys.stderr, "python %s <blast.kegg.anno>" % (sys.argv[0])

filtPathwayByOrg.py文件
#! /usr/bin/env python
import sys
import re
import os

def main(ref_pathway, pathway):
    ref_dic = {}
    ref_name = os.path.basename(ref_pathway)
    for line in open(ref_pathway):
        line = line.rstrip()
        lines = line.split()
        pathway_id = lines[0].split(":")[-1][len(ref_name):]
        ref_dic[pathway_id] = 0
        #print pathway_id

    for line in open(pathway):
        line = line.rstrip()
        lines = line.split()
        if re.search("^#", line):
            print line
            continue
        if lines[0] not in ref_dic:
            continue
        print line

if len(sys.argv) > 1:
    main(sys.argv[1], sys.argv[2])
else:
    print >>sys.stderr, "python %s <refPathway> <pathway>" % (sys.argv[0])


listPathway/osa文件格式
path:osa00010   Glycolysis / Gluconeogenesis - Oryza sativa japonica (Japanese rice) (RefSeq)
path:osa00020   Citrate cycle (TCA cycle) - Oryza sativa japonica (Japanese rice) (RefSeq)
path:osa00030   Pentose phosphate pathway - Oryza sativa japonica (Japanese rice) (RefSeq)

Pathway2Plot.py文件
#! /usr/bin/env python
import sys
import re
sys.path.append("/hellogene/scgene01/user/chenjiehu/bin/module/")
import pipeline

def main(pathway, outdir):
    pipeline.mkdir(outdir)
    keg = "/hellogene/scgene02/RD/RNASeq/db/KEGG/br08901.keg"
    pathway_dic = {}
    for line in open(pathway):
        line = line.rstrip()
        if re.search("^#", line):
            continue
        lines = line.split("\t")
        pid = lines[0].split()[0]
        genes = lines[-1].split(";")
        if pid not in pathway_dic:
            pathway_dic[pid] = []
        pathway_dic[pid].extend(genes)

    keg_dic = {}
    key = ""
    key1 = ""
    for line in open(keg):
        line = line.rstrip()
        lines = line.split()
        if re.search("^A", line):
            key = line[4:-4]
            keg_dic[key] = {}
            continue
        if re.search("^B", line):
            key1 = " ".join(lines[1:])
            if key1 not in keg_dic[key]:
                keg_dic[key][key1] = {}
            #print key1
            continue
        key2 = ""
        if re.search("^C", line):
            key2 = lines[1]
        if key2 in pathway_dic:
            for i in pathway_dic[key2]:
                keg_dic[key][key1][i] = 0
        #print line

    factors = []
    OUT = open("%s/pathway2plot.xls" % (outdir), 'w')
    OUT.write("RootClassification\tSecondClassification\tGeneNum\n")
    for i in sorted(keg_dic):
        for j in sorted(keg_dic[i]):
            if len(keg_dic[i][j]) > 0:
                OUT.write("%s\t%s\t%d\n" % (i, j, len(keg_dic[i][j])))
                factors.append(j)
    OUT.close()

    plot_r = "%s/pathway2plot.R" % (outdir)
    OUT = open(plot_r, 'w')
    OUT.write('library("ggplot2")\n')
    OUT.write('pathway2plot <- read.table("%s/pathway2plot.xls", header = T, sep = "\\t")\n' % (outdir))
    OUT.write('pathway2plot$SecondClassification = factor(pathway2plot$SecondClassification, levels=c(\'%s\'))\n' %\
            ("', '".join(factors)))
    OUT.write('pdf(file = "%s/pathwaySta.pdf", 12, 8)\n' % (outdir))
    OUT.write('ggplot(data=pathway2plot, aes(x=SecondClassification, y=GeneNum, fill=RootClassification)) + geom_bar(stat="identity") + coord_flip() + labs(y = "Number of genes", x = "")\n')
    OUT.write('dev.off()\n')
    OUT.close()

    cmd = "/hellogene/scgene01/bio/bin/R/bin/R CMD BATCH %s" % (plot_r)
    pipeline.cmd_work(cmd)

if len(sys.argv) > 1:
    main(sys.argv[1], sys.argv[2])
else:
    print >>sys.stderr, "python %s <pathway.anno> <outdir>" % (sys.argv[0])

kegg.pathway.filt文件
#pathwayID      pathway koNum   geneNum genes
00010   Glycolysis / Gluconeogenesis    32      128     LOC_Os05g44922;LOC_Os04g39420;LOC_Os06g05860;LOC_Os01g53680;LOC_Os01g09570;LOC_Os05g10650;LOC_Os09g24910;LOC_Os08g34050;LOC_Os10g26570;LOC_Os09g20820;LOC_Os03g15950;LOC_Os03g14450;LOC_Os10g08550;LOC_Os06g04510;LOC_Os09g26880;LOC_Os06g22060;LOC_Os08g25720;LOC_Os06g13810;LOC_Os02g48360;LOC_Os09g12650;LOC_Os11g10480;LOC_Os11g10510;LOC_Os01g67860;LOC_Os05g33380;LOC_Os08g02700;LOC_Os01g02880;LOC_Os06g40640;LOC_Os11g07020;LOC_Os06g14510;LOC_Os03g56460;LOC_Os09g29070;LOC_Os08g37380;LOC_Os12g05110;LOC_Os07g08340;LOC_Os11g05110;LOC_Os11g10980;LOC_Os04g58110;LOC_Os03g46910;LOC_Os01g16960;LOC_Os03g20880;LOC_Os10g42100;LOC_Os01g47080;LOC_Os12g08170;LOC_Os06g01630;LOC_Os08g33440;LOC_Os07g22720;LOC_Os06g30460;LOC_Os02g01500;LOC_Os04g40950;LOC_Os02g38920;LOC_Os02g07490;LOC_Os08g03290;LOC_Os06g45590;LOC_Os08g34210;LOC_Os10g11140;LOC_Os03g50480;LOC_Os02g51590;LOC_Os02g32490;LOC_Os04g33190;LOC_Os02g01510;LOC_Os06g01590;LOC_Os03g60370;LOC_Os05g09500;LOC_Os01g52450;LOC_Os05g44760;LOC_Os01g09460;LOC_Os01g53930;LOC_Os07g26540;LOC_Os07g09890;LOC_Os01g71320;LOC_Os05g45590;LOC_Os01g46950;LOC_Os09g15820;LOC_Os02g08030;LOC_Os08g14330;LOC_Os04g51390;LOC_Os04g56290;LOC_Os05g49430;LOC_Os05g36270;LOC_Os01g64660;LOC_Os06g45370;LOC_Os03g16050;LOC_Os03g15050;LOC_Os06g45710;LOC_Os01g58610;LOC_Os05g41640;LOC_Os02g07260;LOC_Os10g07229;LOC_Os11g10520;LOC_Os09g36450;LOC_Os01g05490;LOC_Os01g62420;LOC_Os06g13720;LOC_Os02g50620;LOC_Os04g02900;LOC_Os03g44300;LOC_Os12g42230;LOC_Os08g42410;LOC_Os09g33500;LOC_Os06g15990;LOC_Os04g45720;LOC_Os12g07810;LOC_Os11g08300;LOC_Os02g49720;LOC_Os02g43194;LOC_Os02g43280;LOC_Os04g38540;LOC_Os04g38530;LOC_Os09g02140;LOC_Os10g06720;LOC_Os02g36600;LOC_Os01g06660;LOC_Os05g39320;LOC_Os05g39310;LOC_Os03g18220;LOC_Os07g49250;LOC_Os07g02030;LOC_Os01g22520;LOC_Os05g06460;LOC_Os01g23610;LOC_Os05g06750;LOC_Os07g42924;LOC_Os02g57040;LOC_Os03g08999;LOC_Os03g09020;LOC_Os08g37140;LOC_Os05g40420;LOC_Os01g60190

**********************************************************************************
下载URL文件和通路图片文件
#kegg.blastp.out由上面kegg的步骤产生,RNASeq为KEGG处理前的原始的每一个文件夹，edgeR.29736.dir选择时间最新的一个，29736数字越大月新，或者ll查看修改时间确定，gene_count_matrix.csv.DYM_vs_TYM.edgeR.DE_results在该文件夹下，产生kegg.url文件
1.python /hellogene/scgene02/RD/RNASeq/bin/KEGG/colorPathway.py kegg.blastp.out ../RNASeq/edgeR.29736.dir/gene_count_matrix.csv.DYM_vs_TYM.edgeR.DE_results > kegg.url
colorPathway.py文件：
#! /usr/bin/env python
import sys
import re

def main(blast_hit_anno, de_express):
    kid2pathway = "/hellogene/scgene02/RD/RNASeq/db/KEGG/KID2Pathway.list"
    pathway_anno = "/hellogene/scgene02/RD/RNASeq/db/KEGG/pathway.list"
    pathway_anno_dic = pathwayInfo(pathway_anno)
    express_dic = deExpress(de_express)

    kid_dic = {}
    for line in open(blast_hit_anno):
        line = line.rstrip()
        if re.search("^#", line):
            continue
        lines = line.split()
        gid = lines[0]
        kid = lines[1].split("(")[0]
        if kid not in kid_dic:
            kid_dic[kid] = []
        kid_dic[kid].append(gid)

    pathway_dic = {}
    for line in open(kid2pathway):
        line = line.rstrip()
        lines = line.split()
        map_id = lines[0].split(":")[1][3:]
        kid = lines[1].split(":")[-1]
        #print map_id, kid
        if kid in kid_dic:
            if map_id not in pathway_dic:
                pathway_dic[map_id] = {}
            pathway_dic[map_id][kid] = kid_dic[kid]


    for i in sorted(pathway_dic):
        unigene_dic = {}
        unigenes = []
        ko_dic = {}
        colors = []
        for j in pathway_dic[i]:
            ko_dic[j] = {}
            for k in pathway_dic[i][j]:
                if k in express_dic:
                    express_up_down = express_dic[k]
                    ko_dic[j][express_up_down] = 0

            color_set = "grey"
            if len(ko_dic[j]) == 2:
                color_set = "yellow"
            elif "up" in ko_dic[j]:
                color_set = "red"
            elif "down" in ko_dic[j]:
                color_set = "green"
            elif len(ko_dic[j]) > 0:
                print >>sys.stderr, "color set error", i, j, ko_dic[j]
                sys.exit(1)
            colors.append("%s%%09%s" % (j, color_set))
        print "#%s" % (pathway_anno_dic[i])
        print "http://www.kegg.jp/kegg-bin/show_pathway?map%s/%s" % (i, \
                "/".join(colors))


def pathwayInfo(pathway_anno):
    pathway_anno_dic ={}
    for line in open(pathway_anno):
        line = line.rstrip()
        lines = line.split("\t")
        map_id = lines[0].split(":")[1][3:]
        map_anno = lines[1]
        pathway_anno_dic[map_id] = map_anno

    return pathway_anno_dic


def deExpress(de_express):
    express_dic = {}
    log_fc_cloum = 0
    head = 0
    for line in open(de_express):
        line = line.rstrip()
        lines = line.split()
        if head == 0:
            for i in xrange(len(lines)):
                if lines[i] == "logFC":
                    log_fc_cloum = i
                head = 1
            continue
        if log_fc_cloum == 0:
            print >>sys.stderr, "not found logFC in express file title"
            sys.exit(1)

        gene_express = float(lines[log_fc_cloum])
        express_dic[lines[0]] = "up"
        if gene_express < 0:
            express_dic[lines[0]] = "down"
    return express_dic


if len(sys.argv) > 1:
    main(sys.argv[1], sys.argv[2])
else:
    print >>sys.stderr, "python %s <blast.kegg.anno> <diffExpress>" % (sys.argv[0])



#kegg.blastp.out由上面kegg的步骤产生,kegg.url由上一步产生的URL文件
2.python /hellogene/scgene02/RD/RNASeq/bin/keggMapGet.py  kegg.blastp.out kegg.url
keggMapGet.py文件：
#! /usr/bin/env python
import sys
import re
import os

def main(kegg_anno, mark_url):
    ko_dic = {}
    for line in open(kegg_anno):
        line = line.rstrip()
        if re.search("^#", line):
            continue
        lines = line.split()
        kid = lines[1].split("(")[0]
        if kid not in ko_dic:
            ko_dic[kid] = []
        ko_dic[kid].append(lines[0])

    name_tag = ""
    for line in open(mark_url):
        line = line.rstrip()
        if re.search("^#", line):
            name_tag = line[1:]
            continue
        url = line
        name_tag = name_tag.replace("/", "or")
        name_tag = name_tag.replace(" ", "_")
        name_tag = name_tag.replace("(", "-")
        name_tag = name_tag.replace(")", "-")
        name_tag = name_tag.replace(",", "-")
        name_tag = name_tag.replace("'", "-")

        cmd = "mkdir %s" % (name_tag)
        os.system(cmd)

        cmd = "wget --convert-links -O %s/%s.htm\t\"%s\"" % (name_tag, name_tag, url)
        os.system(cmd)

        html_file = "%s/%s.htm" % (name_tag, name_tag)

        for line in open(html_file):
            line = line.rstrip()
            if re.search("www.kegg.jp", line) and re.search("png", line):
                png_url = line.split()[1].split('"')[1]
                png_name = png_url.split("/")[-1]
                cmd = "wget -O  %s/%s \"%s\"" % (name_tag, png_name, png_url)
                #print cmd
                os.system(cmd)

        key = 0
        out = "%s.html" % (html_file.split(".")[0])
        OUT = open(out, 'w')
        for line in open(html_file):
            line = line.rstrip()
            if re.search("button_Hb.gif", line):
                continue
            if re.search("</table>", line) and key == 0:
                key = 1
                OUT.write("%s\n" % (line))

            if re.search("pathwayimage", line):
                key = 0
                lines = line.split()
                lines[1] = 'src="%s"' % (lines[1].split('"')[-2].split("/")[-1])
                line = " ".join(lines)

            if key == 1:
                continue

            if re.search("K[0-9]", line):
                new_line = line
                for i in xrange(1, len(line) - 1):
                    if (line[i-1] == '"' or line[i-1] == ' ') and \
                            line[i] == "K" and \
                            re.search("[0-9]", line[i+1]):
                        j = i
                        while True:
                            j += 1
                            if not re.search("[0-9]", line[j]):
                                break
                        ko_id = line[i : j]
                        if ko_id in ko_dic:
                            new_line = new_line.replace("\"%s" % (ko_id),\
                                    "\"%s:%s" % (ko_id, ";".join(ko_dic[ko_id])))
                            new_line = new_line.replace(" %s" % (ko_id),\
                                    " %s:%s" % (ko_id, ";".join(ko_dic[ko_id])))
                line = new_line

            OUT.write("%s\n" % (line))
        OUT.close()

if len(sys.argv) > 1:
    main(sys.argv[1], sys.argv[2])
else:
    print >>sys.stderr, "python %s <KEGG_Annotation.xls> <url>" % (sys.argv[0])
*********************************************************************************
分类并删除无关的
在url文件夹里将其他文件移出，只保留下载的url文件夹
ls > list
python keggMapClass.py list > a.sh
sh a.sh
查看需要的分类，即pathway2plot.xls的第一列
awk -F "\t" '{print $1}' ../kegg/pathway2plot.xls
将唔需要的文件夹删除
rm -rf Human_
rm -rf Ch

keggMapClass.py文件：
#! /usr/bin/env python
import sys
import re
import os


def main(keggmap_list):
    keg = "/hellogene/scgene02/RD/RNASeq/db/KEGG/br08901.keg"
    keggmap_dic = {}
    for keggmap in open(keggmap_list):
        keggmap = keggmap.rstrip()
        map_name = os.path.basename(keggmap)
        keggmap_dic[map_name] = keggmap

    keg_dic = {}
    key = ""
    key1 = ""
    for line in open(keg):
        line = line.rstrip()
        lines = line.split()
        if re.search("^A", line):
            key = line[4:-4]
            keg_dic[key] = {}
            continue
        if re.search("^B", line):
            key1 = " ".join(lines[1:])
            if key1 not in keg_dic[key]:
                keg_dic[key][key1] = []
            #print key1
            continue
        key2 = ""
        if re.search("^C", line):
            key2 = " ".join(lines[2:])
            key2 = key2.replace("/", "or")
            key2 = key2.replace(" ", "_")
            key2 = key2.replace("(", "-")
            key2 = key2.replace(")", "-")
            key2 = key2.replace(",", "-")
            key2 = key2.replace("'", "-")
        if key2 in keggmap_dic:
            keg_dic[key][key1].append(keggmap_dic[key2])

    for i in sorted(keg_dic):
        for j in sorted(keg_dic[i]):
            if len(keg_dic[i][j]) > 0:
                x = i.replace(" ", "_")
                y = j.replace(" ", "_")
                y = y.replace(":", "")
                print "mkdir -p %s/%s" % (x, y)
                for k in keg_dic[i][j]:
                    print "mv %s %s/%s" % (k, x, y)


if len(sys.argv) > 1:
    main(sys.argv[1])
else:
    print >>sys.stderr, "python %s <pathway.list>" % (sys.argv[0])

==================================================================================
巴戟天寡糖结合
根据Trinity.fasta和nr.Viridiplantae.fa产生01Unigene，02Unigene2prot，03UnigeneNProt结果
python /hellogene/scgene01/user/huangjingchuan/bin/getSeqFromNr.py /hellogene/scgene02/RD/RNASeq/db/nr/nr /hellogene/scgene02/RD/RNASeq/db/ftp.ncbi.nlm.nih.gov/pub/taxonomy/accession2taxid/prot.accession2taxid.gz  /hellogene/scgene02/RD/RNASeq/db/taxdump/taxidlineage.dmp 33090 > nr.Viridiplantae.fa
#建库，getSeqFromNr.py复制过来才能修改，nr在那边是nr.gz，需要解压过来，gzip -dc /hellogene/scgene02/RD/RNASeq/db/nr/nr.gz > ./nr (-d为解压缩，-c为标准输出且保留源文件，所以需要 > 和 ./nr),33090为NCBI-搜索巴戟天拉丁名Morinda officinalis-Taxonomy-Gynochthodes officinalis-Viridiplantae(植物界）-Viridiplantae-Taxonomy ID: 33090得到）

each.sh
#uniqueGene产生01Unigene文件夹及下面unigene.fa文件
python /hellogene/scgene02/RD/RNASeq/bin/uniqueGene.py Trinity.fasta 01Unigene
#nr.Viridiplantae.fa为上一步建立的库，01Unigene/unigene.fa为上一步产生的结果
python /hellogene/scgene02/RD/RNASeq/bin/unigene2prot.py 01Unigene/unigene.fa 1 /hellogene/scgene02/RD/RNASeq/db/index/uniprot_plants.fa ../../db/nr.Viridiplantae.fa 02Unigene2prot 
#02Unigene2prot/prot.fa为上一步产生的结果
python /hellogene/scgene02/RD/RNASeq/bin/unigene2prot2.py 01Unigene/unigene.fa 02Unigene2prot/prot.fa 03UnigeneNProt

----------------------------------------------------------------------


==================================================================================
1.samtools
sudo apt install samtools

2.samstat
安装
tar -zxcf samstat.tgz 
cd samstat
make
sudo make install
使用
samstat <file.sam>  <file.bam>  <file.fa>  <file.fq> .... 

For each input file SAMStat will create a single html page named after the input file name plus a dot html suffix.

3.flexbar
sudo apt install flexbar

4.fastqc
sudo apt install fastqc
===========================================================================
在母中寻子并输出子所在的母的行，VLOOKUP

import sys
  
def main(s,m):
    dic={}
    for line in open(s):
        line=line.strip()
        lines=line.split("\t")
        dic[lines[0]]=0

    for line in open(m):
        line=line.strip()
        lines=line.split()
        if lines[0] in dic:
            print line
main(sys.argv[1],sys.argv[2])

============================================================================
shell教程

1.删除fasta格式文件后1/2的行
num=`grep ">" $1|wc -l`
numa=`expr $num + 1`
sed ''"$numa"',$d' -i $1

awk '{print $1}' out.blastp |sort |uniq > list
awk '{print  $1}' list |while read a ;do grep $a out.blastp |head -n 1 ;done |sort -nk 3 |less -S
less /hellogene/scgene02/RD/RNASeq/db/KEGG/GeneBySpecies/nta.fa

列出目录的方式
1.ls -F | grep '/$'
ls的-F 使得文件夹后面显示/符号，然后grep 查找即可
2.ls -l | grep '^d'|awk '{print $NF}'
ls -l显示详细信息，grep 查找^d表示行首为d即director的文件夹，awk输出最后一列的文件夹名，推荐使用这个命令
3.ls -ap | grep '/'
ls的-ap可以使得文件夹显示/符号，grep查找
4.tree -d -L 1
tree显示树状文件结构，-d只显示目录，-L选择目录的深度后接参数

===========================================================================
构建进化树
1.下载多个科多个属下的种的基因组文件，选择时需要基因序列数大于60(80）
2.每个科可以选多个属，每个属由于构建进化树只能选一个物种，同属的物种进化树无太大意义
3.将所有的合格的基因组文件放在同意目录下，重命名为前几位id
重命名如把aa.result.fasta > aa.fasta
ls * > list
vim list -> :%s/.result.fasta//g (删除.result.fasta序列)
cat list|while read a;do cp $a".result.fasta" newdir/$a".fasta" ;done
4.如果基因组文件的id为>NODE_1_length_1946_cov_3.01319_ID_1_arhgap35需要取>arhgap35作为id
faid.py
import sys
def main(fa):
    for line in open(fa,'r'):
        if line[0]==">" :
            line=line.strip().split("_")
            print ">"+line[8]
        else:
            line=line.strip()
            print line
main(sys.argv[1])

5.从基因组文件id为arhgap35的文件中提取基因序列，重定向累积输出多个同类基因不同物种的基因文件
ls ../*.fa |while read a;do python ../../choice_gene.py ../allgene.list $a;done

list3里是fasta文件里的id如arhgap35共103种
ankrd11.fa   fbxl14b.fa   lig4.fa          ptchd4.fa            stox2a.fa
ANKRD50.fa   fem1a.fa     LOC100333177.fa  pwwp2b.fa            syne1b.fa
apc2.fa      fign.fa      LOC100537710.fa  rag1.fa              tanc2b.fa
arhgap35.fa  filip1l.fa   LOC572016.fa     RAG2.fa              tenm3.fa
ash1l.fa     flrt2.fa     LOC794462.fa     rassf10b.fa          tmem151b.fa
b3galt2.fa   frmpd3.fa    LOC799850.fa     rnf213.fa            trip11.fa
bach2b.fa    ftr15.fa     mab21l2.fa       ror2.fa              tshz1.fa
bcl11ba.fa   fzd7a.fa     map1ab.fa        SACS.fa              TTN.fa
bend3.fa     gpatch8.fa   mogs.fa          sall4.fa             tubb2.fa
ccdc177.fa   GPER.fa      mycbp2.fa        scn8ab.fa            tulp4a.fa
celsr2.fa    grin2aa.fa   nkrf.fa          sema6e.fa            vcpip1.fa
chpfa.fa     hivep1.fa    nod2.fa          sh3bp4.fa            zbed4.fa
chsy1.fa     hsp70l.fa    odz4.fa          si:ch211236k19.7.fa  zc3h12b.fa
cilp.fa      igsf9bb.fa   otud7b.fa        si:dkey151g22.1.fa   zeb2b.fa
cnr1.fa      iqsec1.fa    pcdh11.fa        skor1b.fa            zgc:103511.fa
dopey1.fa    kcna2b.fa    pdzrn3b.fa       slc18a3a.fa          zic5.fa
dsela.fa     kcnd2.fa     plagl2.fa        slc8a4a.fa           znf319.fa
elfn1.fa     kcnf1a.fa    pomgnt2.fa       slitrk3.fa           znf536.fa
enc1.fa      kiaa2022.fa  pou3f2b.fa       slitrk4.fa
extl3.fa     klhl11.fa    ppl.fa           socs6.fa
FAT1.fa      lats2.fa     prdm11.fa        specc1la.fa



choice_gene.py

#! /usr/bin/python

import sys
from Bio import SeqIO
clean={
'CO1':'COX1',
'CO2':'COX2',
'CO3':'COX3',
'CYTOCHROME C OXIDASE SUBUNIT 1':'COX1',
'CYTOCHROME C OXIDASE SUBUNIT I':'COX1',
'CYTOCHROME C OXIDASE SUBUNIT 2':'COX2',
'CYTOCHROME C OXIDASE SUBUNIT II':'COX2',
'CYTOCHROME C OXIDASE SUBUNIT 3':'COX3',
'CYTOCHROME C OXIDASE SUBUNIT III':'COX3',
'COX2A':'COX2',
'COI':'COX1',
'COII':'COX2',
'COIII':'COX3',
'COXI':'COX1',
'COXII':'COX2',
'COXIII':'COX3',
'cytb':'CYTB',
'CYTOCHROME B':'CYTB',
'ATPASE 9':'ATP9',
'ATPASE9':'ATP9',
'ATPASE9':'ATP9',
'ATPASE 6':'ATP6',
'ATPASE6':'ATP6',
'ATP SYNTHASE F0 SUBUNIT 6':'ATP6',
'atp9':'ATP9',
'atp6':'ATP6',
'atp8':'ATP8',
'atp 8':'ATP8',
'ATPASE 8':'ATP8',
'ATP 8':'ATP8',
'cytB':'CYTB',
'cob':'CYTB',
'COB':'CYTB',
'CYT B':'CYTB',
'CYT B':'CYTB',
'NADH1':'ND1',
'NADH DEHYDROGENASE SUBUNIT 1':'ND1',
'NADH2':'ND2',
'NADH DEHYDROGENASE SUBUNIT 2':'ND2',
'NADH3':'ND3',
'NADH DEHYDROGENASE SUBUNIT 3':'ND3',
'NADH4':'ND4',
'NADH DEHYDROGENASE SUBUNIT 4':'ND4',
'NADH4L':'ND4L',
'NADH DEHYDROGENASE SUBUNIT 4L':'ND4L',
'NADH5':'ND5',
'NADH DEHYDROGENASE SUBUNIT 5':'ND5',
'NADH6':'ND6',
'NADH DEHYDROGENASE SUBUNIT 6':'ND6',
'NAD1':'ND1',
'NAD2':'ND2',
'NAD3':'ND3',
'NAD4':'ND4',
'NAD4L':'ND4L',
'NAD5':'ND5',
'NAD6':'ND6',
}
def getGene(gene):
	if gene in clean.keys():
		name=clean[gene]
	if gene not in clean.keys():
		name=gene
	return name
def main(gene_file,cds_file):
	name=cds_file.split('/')[-1].split('.')[0]
	gene_list=[]
	have=[]
	for i in open(gene_file,'r'):
		gene_name=i.strip()
		gene_list.append(gene_name)
	for i in SeqIO.parse(cds_file,'fasta'):
		gene_name=i.id
		new_name=getGene(gene_name)
		if new_name in gene_list and new_name not in have:
			out=open(new_name+'.fa','a')
			have.append(new_name)
			out.write('>%s\n%s\n' % (name,i.seq))
			out.close()
if len(sys.argv) ==3 :
	main(sys.argv[1],sys.argv[2])
if len(sys.argv) !=3:
	print 'python %s <gene.list> <fa>' % sys.argv[0]

7.同最先的建树一样，开始muscle比对
ls *.fa |while read a;do muscle -in $a -out $a.ms;done
6.由于每种基因里物种的数量不同（每个物种不一定含有所有基因），为了让每个基因文件在比对后都有所有物种的基因序列，没有的也为---填充
mkdir ms
mv *.ms ms
ls *.ms |while read a;do python lack.py id.list $a > $a.lack;done
注意：由于比对后的序列的id是没有小数点及后的数字，所以id.list注意要保证没有小数的且和比对后的序列里的id一致
lack.py
#1 /usr/bin/python

import sys
from Bio import SeqIO

def main(accession,fa):
        dic={}
        for i in SeqIO.parse(fa,'fasta'):
                dic[i.id]=i.seq
                length=len(i.seq)
        for i in open(accession,'r'):
                line=i.strip().split()
                name=line[0]
                if name in dic.keys():
                        print '>%s\n%s' % (name,dic[name])
                if name not in dic.keys():
                        print '>%s\n%s' % (name,'-'*length)
if len(sys.argv) ==3 :
        main(sys.argv[1],sys.argv[2])
if len(sys.argv) !=3:
        print 'python %s <accession.list> <fa>' % sys.argv[0]

产生基因属数都为最大数量的fasta文件
8.产生cds.fsa文件
# -*- coding: utf-8 -*-
'''''''''''''''''''''''''''''''''''''''''''''''''''
在获取共同基因后（一些*.fa.ms文件），再将其拷贝至一处，在此目录下先运行
cp *.lack ../
cd ../
ls *.fa|while read a;do mv $a.ms.lack $a.ms;done 
#ls *.ms| sort -n |cut -d '.' -f 1 > gene.list
#grep '>' *.ms| cut -d '>' -f 2 |cut -d '-' -f 1|cut -d '.' -f 1| sort -u > id.list
注意id.list的id没有小数点的
这里的ms文件其实是lack文件，因为下面的代码识别的是.ms的文件
注：这里genenum.list（是此次所有物种产生的所有的基因，不是用于选择的所有的基因）和id.list也可以自己指定，其顺序即最后生成的cds.fsa文件中
每个基因的连接顺序以及cds.fsa文件中物种的先后顺序
再在放有*.fa.ms的文件夹下运行本程序
python mscollect.py -g genenum.list -s species.list
运行完生成cds.fsa是我们需要的
Author: lurui@scgene.com
Date: 2016.5.4
'''''''''''''''''''''''''''''''''''''''''''''''''''
import re
from Bio import SeqIO
import sys
import argparse
def main(glist, slist):
        glst = []
        slst = []
        f={}
        result={}
        for i in open(glist, 'r'):
                if not i.startswith(' '):
                        glst.append(i.split()[0])
        for j in open(slist, 'r'):
                if not i.startswith(' '):
                        slst.append(j.split()[0])
        outfile = open('cds.fsa', 'w')
        #for m in slst:
        #       f[m] = open(m+'.fsa', 'w')
        #       print >> f[m], '>%s' %m
        for i in glst:
                ifname = i + '.fa.ms'
                record = SeqIO.parse(ifname, 'fasta')
                for t in record:
                        for j in slst:
                                if re.search(j, t.id):
                                        result.setdefault(j, []).append(str(t.seq))                                      
                                        #print >> f[j], t.seq
        for m in slst:
                print >> outfile, '>%s\n%s' %(m, ''.join(result[m]))
        outfile.close()
        #for m in slst: f[m].close()




if __name__=='__main__':
        if len(sys.argv) == 1:
                print 'Usage: <python> <script> [-h]'
                exit(1)
                
	parser = argparse.ArgumentParser(conflict_handler = 'resolve')
	parser.add_argument('-g', '--genelist', help='all the gene')
	parser.add_argument('-s', '--spelist', help='all the species')	
	args = parser.parse_args()
        main(args.genelist, args.spelist)

9.建模建树

==================================================================
方法 1：

运行下面命令为为当前用户临时设置 HISTTIMEFORMAT 变量。这会一直生效到下次重启。

# export HISTTIMEFORMAT='%F %T '

方法 2：

将 HISTTIMEFORMAT 变量加到 .bashrc 或 .bash_profile 文件中，让它永久生效。

# echo 'HISTTIMEFORMAT="%F %T "' >> ~/.bashrc

或

# echo 'HISTTIMEFORMAT="%F %T "' >> ~/.bash_profile

运行下面命令来让文件中的修改生效。

#source~/.bashrc

或

#source~/.bash_profile 

方法 3：

将 HISTTIMEFORMAT 变量加入 /etc/profile 文件中，让它对所有用户永久生效。

# echo 'HISTTIMEFORMAT="%F %T "' >> /etc/profile

运行下面命令来让文件中的修改生效。

#source/etc/profile

====================================================
提取prot.fa的序列长度和out.blastp的相似度和匹配长度
#! /usr/bin/python

import sys
from Bio import SeqIO
fa_length={}
have={}
for i in SeqIO.parse(sys.argv[1],'fasta'):
    fa_length[i.id]=len(i.seq)
for i in open(sys.argv[2],'r'):
    line=i.strip().split()
    have.setdefault(line[0],0)
    if have[line[0]] == 1 :
        continue
    print "%s\t%s\t%s/%s" % (line[1],fa_length[line[1]],line[2],line[3])
    have[line[0]]+=1

============================================
cd new_RNASeq
awk '{if($2 != 0 && $3 != 0 && $4 != 0 && $5 != 0 &&6 != 0 && $7 != 0)print $1}' ../RNASeq/gene_count_matrix.csv > list
python /hellogene/scgene02/RD/RNASeqRef/P135/04Enrichment/bin/deAnno.py list /hellogene/scgene02/RD/RNASeq/Oryza/all.pep.kegg.blastp.2 > kegg.blastp
python /hellogene/scgene02/RD/RNASeq/bin/geneAnnoKEGG.py kegg.blastp > kegg.blastp.out
python /hellogene/scgene02/RD/RNASeq/bin/keggGene2Pathway.py kegg.blastp.out > kegg.pathway
python /hellogene/scgene02/RD/RNASeq/bin/KEGG/filtPathwayByOrg.py /hellogene/scgene02/RD/RNASeq/db/KEGG/listPathway/osa kegg.pathway > kegg.pathway.filt
mkdir kegg
python /hellogene/scgene02/RD/RNASeq/bin/KEGG/Pathway2Plot.py kegg.pathway.filt kegg


===========================================
平均深度
 # -*- coding: utf-8 -*- 
'''''''''''''''''''''''''''''''''''''''''''''
在多序列的fasta文件remap到fastq后，会产生一个map.dp
的文件，将其拷贝至一处，运行python averdep.py map.dp
会产生一个map.dp.out的文件，这个文件第一列是多序列的fasta
文件中的序列id，第二列是多序列的fasta文件的序列的平均深度
（注：在remap的时候序列描述行中如果有空格remap会只取第一个空格前的部分作为id）
Author: lurui@scgene.com
Date: 2016.4.20
'''''''''''''''''''''''''''''''''''''''''''''
from __future__ import division
import re
import sys
from Bio import SeqIO

def main(depfile):
    outfile = open(depfile+'.out', 'w')
    dep ={}
    dep1={}
    fa_len={}
    for i in SeqIO.parse(fa,'fasta'):
	fa_len[i.id]=len(i.seq)
    for i in open(depfile, 'r'):
	eachlst=i.strip().split()
        #eachlst = re.findall(r'\w+', i)
        seqid = eachlst[0]
        seqdep = int(eachlst[-1])
        dep.setdefault(seqid, []).append(seqdep)
    	dep1.setdefault(seqid,0)
	dep1[seqid]=dep1[seqid]+1
    newdep ={}
    for k, v in dep.items():
        averdep = average(v)
        newdep[k]=[averdep,round(float(dep1[k])/fa_len[k],2)]
    sortdep = sorted(newdep.iteritems(), key = lambda d:d[1], reverse=True)
    print >> outfile, '%s\t%s\t%s' %('Seqid', 'Average_depth','cov')
    for j in sortdep:
        print >> outfile, '%s\t%s\t%s' %(j[0], j[1][0],j[1][1])
    outfile.close()
    
def average(lst):
    num = len(lst)
    summary = sum(lst)
    aver = round(summary / num, 2)
    return aver
    
if __name__=='__main__':
    if len(sys.argv) == 3:
        scgene, depfile, fa= sys.argv
        main(depfile)
    else:
        print 'Usage: <python> <script> <depthfile>'
        exit(2)
======================================================
sudo apt install tophat2
sudo apt install bowtie2
sudo apt install hisat2
sudo apt install cufflinks
转录组本组装（利用Cufflinks），转录本与已有基因组注释比较（利用Cuffcompare）、合并（利用Cuffmerge），转录组本差异表达分析（利用Cuffdiff）。

下载hisat-x64包
tar -zxf ***.gz
sudo mv hisat_dir /usr/local/bin
vim ~/.bashrc
ctrl+G
export PATH="/usr/local/bin/hisat_dir/:$PATH"
source ~/.bashrc
注意:$PATH不可少

RNA-Seq基因组比对工具HISAT2           

HISAT2是TopHat2/Bowti2的继任者，使用改进的BWT算法，实现了更快的速度和更少的资源占用，作者推荐TopHat2/Bowti2和HISAT的用户转换到HISAT2。
官网：
https://ccb.jhu.edu/software/hisat2/index.shtml

建立基因组索引
hisat2-build –p 4 genome.fa genome

建立基因组+转录组+SNP索引：
bowtie2的索引只有基因组序列信息，tophat2比对时，转录组信息通过-G参数指定。HISAT2建立索引时，就应该把转录组信息加进去。
HISAT2提供两个Python脚本将GTF文件转换成hisat2-build能使用的文件：
extract_exons.py Homo_sapiens.GRCh38.83.chr.gtf > genome.exon

extract_splice_sites.py Homo_sapiens.GRCh38.83.chr.gtf > genome.ss

此外，HISAT2还支持将SNP信息加入到索引中，这样比对的时候就可以考虑SNP的情况。这仍然需要将SNP文件转换成hisat2-build能使用的文件：
extract_snps.py snp142Common.txt > genome.snp

最后，将基因组、转录组、SNP建立索引：
hisat2-build -p 4 genome.fa --snp genome.snp --ss genome.ss --exon genome.exon genome_snp_tran

官网提供了人和小鼠的索引文件下载，压缩包有make_grch38_tran.sh文件，详细记录了创建索引的过程。
运行HISAT2
hisat2 -p 16 -x ./grch38_tran/genome_tran -1 SRR534293_1.fastq -2 SRR534293_2.fastq –S SRR534293.sam

-x 指定基因组索引
-1 指定第一个fastq文件
-2 指定第二个fastq文件
-S 指定输出的SAM文件
更多参数请查看HISAT2的操作手册：
https://ccb.jhu.edu/software/hisat2/manual.shtml

一个亮点是有一个选项可以直接连接到SRA数据库  这省了不小功夫
–sra-acc <SRA accession number>
输入SRA登录号，比如SRR353653，SRR353654。多组数据之间使用逗号分隔。HISAT将自动下载并识别数据类型，进行比对。

但是也面临不少问题：
1.数据下载速度？效率问题？
2.SRA解压的是raw reads 会有低质量reads和adapter混进去  如何过滤？


/hellogene/scgene02/RD/RNASeqRef/bin/hisat2-2.1.0/hisat2-build -p 32 /hellogene/scgene01/DataBase/Special/Oryza_sativa/Japonica/msu7.0/all.con TYM1_remap/genome_hisat
/hellogene/scgene02/RD/RNASeqRef/bin/hisat2-2.1.0/hisat2 -p 32 --dta -5 20 -3 20 -x TYM1_remap/genome_hisat -1 /hellogene/scgene03/prj/SCG-80530-258/01.fq/TYM1_RRA126335-V_1.clean.fq.gz.fq -2 /hellogene/scgene03/prj/SCG-80530-258/01.fq/TYM1_RRA126335-V_2.clean.fq.gz.fq |/hellogene/scgene01/bio/bin/samtools view -SbF 0x804 - | /hellogene/scgene01/bio/bin/samtools sort -@ 25 - > TYM1_remap/hisat2.srt.bam
/hellogene/scgene02/RD/RNASeqRef/bin/stringtie-1.3.3b.Linux_x86_64/stringtie -G /hellogene/scgene01/DataBase/Special/Oryza_sativa/Japonica/msu7.0/all.gff3 -l T2 -o TYM1_remap/TYM1.gtf -e -p 8 -A TYM1_remap/TYM1.xls -B TYM1_remap/hisat2.srt.bam


===========================================
stringtie安装
下载stringtie-1.3.4d.Linux_x86_64.tar.gz
tar zxf stringtie-1.3.4d.Linux_x86_64.gz
sudo mv stringtie-1.3.4d.Linux_x86_64 /usr/local/bin
echo PATH="/usr/local/bin/stringtie-1.3.4d.Linux_x86_64/:$PATH" >> ~/.bashrc
source ~/.bashrc
一定注意echo 后是 >>追加而非> 覆盖

======================================
安装ballgown
source("http://bioconductor.org/biocLite.R")
biocLite("ballgown")

=--==============================================
MECAT  mecat.sh文件
#$ -S /bin/sh
/hellogene/scgene01/bio/toinstall/MECAT-master/Linux-amd64/bin/mecat2pw -j 0 -k 1 -g 1 -d 01.fq/all.fastq -o SD.fa.pm.can -w MECAT -n 50 -t 32 -a 1 
/hellogene/scgene01/bio/toinstall/MECAT-master/Linux-amd64/bin/mecat2cns -i 0 -t 32 SD.fa.pm.can 01.fq/all.fastq -a 400 -c 3 -l 10000 MECAT_SD.fa
/hellogene/scgene01/bio/toinstall/MECAT-master/Linux-amd64/bin/extract_sequences MECAT_SD.fa MECAT_SD_25x.fasta 20000000 25
/hellogene/scgene01/bio/toinstall/MECAT-master/Linux-amd64/bin/mecat2canu -trim-assemble -p SD -d SD genomeSize=20000000 ErrorRate=0.3 maxMemory=100 maxThreads=32 useGrid=0 Overlapper=mecat2asmpw -pacbio-corrected MECAT_SD_25x.fasta.fasta

https://github.com/xiaochuanle/MECAT#S-installation


    Step 1, using mecat2pw to detect overlapping candidates

mecat2pw -j 0 -d ecoli_filtered.fastq -o ecoli_filtered.fastq.pm.can -w wrk_dir -t 16

    Step 2, correct the noisy reads based on their pairwise overlapping candidates.

mecat2cns -i 0 -t 16 ecoli_filtered.fastq.pm.can ecoli_filtered.fastq corrected_ecoli_filtered

    Step 3, extract the longest 25X corrected reads

extract_sequences corrected_ecoli_filtered.fasta corrected_ecoli_25x.fasta 4800000 25

    Step 4, assemble the longest 25X corrected reads using mecat2cacu

mecat2canu -trim-assemble -p ecoli -d ecoli genomeSize=4800000 ErrorRate=0.02 maxMemory=40 maxThreads=16 useGrid=0 O



/etc/profile
PATH=$PATH:/hellogene/scgene01/bio/bin/MECAT/Linux-amd64/bin/
export PATH
PATH=$PATH:/hellogene/scgene01/bio/bin/DEXTRACTOR/
export PATH
CFLAGS="-I/hellogene/scgene01/bio/include/"
export CFLAGS
LDFLAGS="-L/hellogene/scgene01/bio/lib/"
export LDFLAGS

===========================================================
kmer.sh
#$ -S /bin/sh
mkdir K96
/hellogene/scgene03/RD/RADPlus/bin/SOAPdenovo2-bin-LINUX-generic-r240/SOAPdenovo-127mer all -s lib.conf -o K96/genome -K 96 -p 32
mkdir K127
/hellogene/scgene03/RD/RADPlus/bin/SOAPdenovo2-bin-LINUX-generic-r240/SOAPdenovo-127mer all -s lib.conf -o K127/genome -K 127 -p 32


remap.sh文件
#$ -S /bin/sh
/hellogene/scgene01/bio/bin/bwa index all.cdna
/hellogene/scgene01/bio/bin/bwa mem -t 14 all.cdna  ../../01.fq/DYM2_RRA126333-V_1* ../../01.fq/DYM2_RRA126333-V_2* | /hellogene/scgene01/bio/bin/samtools view -SbF 0x804 - |/hellogene/scgene01/bio/bin/samtools sort -@ 8 - > genome.map.bam
/hellogene/scgene01/bio/bin/samtools index genome.map.bam
/hellogene/scgene01/bio/bin/samtools depth genome.map.bam > map.dp
====================================================================

首先，我们应该做一下质控。如果质控不合格，就需要一些处理，比如去接头、去除量的reads。

（1）去除测序数据中的接头(用到的是fastx_toolkit里面的fastx_clipper工具)：


Usage: fastx_clipper [-h] [-a ADAPTER] [-D] [-l N] [-n] [-d N] [-c] [-C] [-o] [-v] [-z] [-i INFILE] [-o OUTFILE]  #去掉接头序列

 [-a ADAPTER] =接头序列（默认为CCTTAAGG）

 [-l N]       = 忽略那些碱基数目少于N的reads，默认为5

 [-d N]       = 保留接头序列后的N个碱基默认  -d 0

 [-c]         = 放弃那些没有接头的序列.

 [-C]         = 只保留没有接头的序列.

 [-k]         = 报告只有接头的序列.

 [-n]         = 保留有N多序列，默认不保留

 [-v]         =详细-报告序列编号

 [-z]         =压缩输出.

 [-D]       = 输出调试结果.

 [-M N]   =要求最小能匹配到接头的长度N，如果和接头匹配的长度小于N不修剪

 [-i INFILE]  = 输入文件

 [-o OUTFILE] = 输出文件

Example: fastx_clipper -a AGATCGGAAGAGCACACG -l 25 -d 0 -Q 33 -i SRR306394_1.fastq -o SRR306394_1_trimmed.fastq
(注：-l参数，即忽略那些碱基数目少于N的reads。默认的是5，而对于tophat来说，如果reads长度小于20，就会发出警告。因此，考虑到充分利用这些测序数据同时不损失比对的特异性，这里-l参数，取值为25。-Q参数，对于illumina测序来说，这个参数也是必须要加的，因为illumina测序在给单个碱基作质量打分的时候，加上了33，然后才转成的ASC II, 因此在这里需要加 -Q 33. 否则就会报这样的错误“fastx_clipper: Invalid quality score value (char '#' ord 35 quality value -29) on line 4.”)


（2）去除测序数据中的低质量reads(用到的是fastx_toolkit里面的fastq_quality_filter工具)：
Usage:fastq_quality_filter [-h] [-v] [-q N] [-p N] [-z] [-i INFILE] [-o OUTFILE]过滤低质量序列

 [-q N]       = 最小的需要留下的质量值

 [-p N]       = 每个reads中最少有百分之多少的碱基需要有-q的质量值

 [-z]         =压缩输出

 [-v]       =详细-报告序列编号，如果使用了-o则报告会直接在STDOUT，如果没有则输入到STDERR

Example: fastq_quality_filter -q 20 -p 80 -Q 33 -i SRR306394_1.fastq -o SRR306394_1_filtered.fastq
(注：-q参数，即是指最小需要留下的质量值，这里需要保留Q20(99.99%)以上的，所以对应的是20。-p参数，即是指每个reads中最少有百分之多少的碱基需要有-q的质量值，这里我们要求的是一条reads中最少有80%以上的碱基质量值达到Q20我们才作保留，因此对应的是80. -Q参数，对于illumina测序来说，这个参数也是必须要加的，因为illumina测序在给单个碱基作质量打分的时候，加上了33，然后才转成的ASC II, 因此在这里需要加 -Q 33. 否则就会报这样的错误“fastx_clipper: Invalid quality score value (char '#' ord 35 quality value -29) on line 4.”)
(注：如果在做fastq_quality_filter之后，QC结果仍然可以看到reads的尾部有低质量的reads,此时可以将-p参数调大，比如调至90或者99)。
========================================================
比对工具

1.TopHat2： –no-coverage-search

http://ccb.jhu.edu/software/tophat/index.shtml

2.STAR: -twopassMode Basic –outFilterType BySJout

https://github.com/alexdobin/STAR/releases

3.HISAT2 2.0.1-beta –dta (or –dta-cufflinks)

http://www.ccb.jhu.edu/software/hisat/index.shtml

4.RASER 0.52 -b 0.03

https://www.ibp.ucla.edu/research/xiao/RASER.html


有参考转录本组装工具

1.Cufflinks 2.2.1 –frag-bias-correct

http://cole-trapnell-lab.github.io/cufflinks/

2.StringTie 1.2.1 -v -B

http://www.ccb.jhu.edu/software/stringtie/


无参考转录本组装工具

1.SOAPdenovoTrans 1.04 -K 25

https://github.com/aquaskyline/SOAPdenovo-Trans/

2.Oases 0.2.09 (Velvetv1.2.10) (velveth haslength: 25) (velvetg options: -read trkg yes)

http://www.ebi.ac.uk/~zerbino/oases/

3. Trinity 2.1.1 –normalize reads

http://trinityrnaseq.sourceforge.net/


三代长read分析工具

1.LoRDEC 0.6 -k 23 -s 3

http://atgc.lirmm.fr/lordec/

2.GMAP 12/31/15 -f 1

http://research-pub.gene.com/gmap/

3. STARlong 2.5.1b

https://github.com/alexdobin/STAR/releases

Followed the recommended options :

–outSAMattributes NH HI NM MD

–readNameSeparator space

–outFilterMultimapScoreRange 1

–outFilterMismatchNmax 2000

–scoreGapNoncan -20

–scoreGapGCAG -4

–scoreGapATAC -8

–scoreDelOpen -1

–scoreDelBase -1

–scoreInsOpen -1

–scoreInsBase -1

–alignEndsType Local

–seedSearchStartLmax 50

–seedPerReadNmax 100000

–seedPerWindowNmax 1000

–alignTranscriptsPerReadNmax 100000

–alignTranscriptsPerWindowNmax 10000

–outSAMstrandField intronMotif

–outSAMunmapped Within

4. IDP 0.1.9

https://www.healthcare.uiowa.edu/labs/au/IDP/


定量工具

1. eXpress 1.5.1 (bowtie2 v2.2.7) (bowtie2 options: -a -X 600 –rdg 6,5 –rfg 6,5 –score-min L,-.6,-.4 –no-discordant –no-mixed)

https://pachterlab.github.io/eXpress/index.html

2. kallisto 0.42.4

http://pachterlab.github.io/kallisto/about.html

3. Sailfish 0.9.0

http://www.cs.cmu.edu/~ckingsf/software/sailfish/

4. Salmon-Aln 0.6.1

https://github.com/COMBINE-lab/salmon

5. Salmon-SMEM 0.6.1

https://github.com/COMBINE-lab/salmon

index: –type fmd

quant: -k,19

6. Salmon-Quasi 0.6.1

https://github.com/COMBINE-lab/salmon

index: –type quasi -k 31

7. featureCounts 1.5.0-p1 -p -B -C

http://subread.sourceforge.net/


差异表达分析工具

1. DESeq2 1.14.1

http://bioconductor.org/packages/release/bioc/html/DESeq2.html

2. edgeR 3.16.5

http://www.bioconductor.org/packages/release/bioc/html/edgeR.html

3. limma 3.30.7

http://bioconductor.org/packages/release/bioc/html/limma.html

4. Cuffdiff 2.2.1

–frag-bias-correct –emit-count-tables

http://cole-trapnell-lab.github.io/cufflinks/

5. Ballgown 2.6.0

https://github.com/alyssafrazee/ballgown

6. sleuth 0.28.1

https://github.com/pachterlab/sleuth


变异分析工具


1. SAMtools 1.2 (bcftools v1.2)

samtools mpileup -C50 -d 100000

https://github.com/samtools/samtools

2. bcftools filter -s LowQual -e ‘%QUAL<20 ——="" dp="">10000’

https://github.com/samtools/bcftools

3.GATK v3.5-0-g36282e4 (picard 1.129)

https://software.broadinstitute.org/gatk/download/

Picard AddOrReplaceReadGroups: SO=coordinate

Picard MarkDuplicates: CREATE INDEX=true VALIDATION STRINGENCY=SILENTGATK

SplitNCigarReads: -rf ReassignOneMappingQuality -RMQF 255 -RMQT 60

-U ALLOW N CIGAR READSGATK

HaplotypeCaller: -stand call conf 20.0

-stand emit conf 20.0 -A StrandBiasBySample

-A StrandAlleleCountsBySampleGATK

VariantFiltration: -window 35 -cluster 3 -filterName FS -filter

“FS >30.0” -filterName QD -filter “QD <2.0”< span="">

RNA编辑


1. GIREMI 0.2.1

https://github.com/zhqingit/giremi

2. Varsim 0.5.1

https://github.com/bioinform/varsim


基因融合

1.FusionCatcher 0.99.5a beta

https://github.com/ndaniel/fusioncatcher

2.JAFFA 1.0.6

https://github.com/Oshlack/JAFFA

3.SOAPfuse 1.27

http://soap.genomics.org.cn/soapfuse.html

4.STAR-Fusion 0.7.0

https://github.com/STAR-Fusion/STAR-Fusion

5.TopHat-Fusion 2.0.14

http://ccb.jhu.edu/software/tophat/fusion_index.shtml
================================================================
sudo apt install canu
 三代组装软件canu学习笔记 (2017-08-07 14:17:43)
转载
▼
	分类： 三代
1：这个组装软件起源于PBcR包含在Celera Assembler中（http://wgs-assembler.sourceforge.net/wiki/index.php/Main_Page），该软件最新版本是8.3之后便不在更新。现在被canu取代。

2：canu(http://canu.readthedocs.io/en/latest/index.html)
参加文献：Koren S, Walenz B P, Berlin K, et al. Canu: scalable and accurate long-read assembly via adaptive k-mer weighting and repeat separation[J]. Genome research, 2017, 27(5): 722-736.

3：目前版本1.5

4：几个重要的参数说明：
minReadLength 用于组装的最短reads,默认1000

corOutCoverage 用于矫正的数据最小coverage，默认是40x,但实际上的数据在30X-35X之间你可以自己设置为50,60,100,当设置为1000，可以用于组装出数据中质粒，一般该参数用于宏基因组组装

contigFilter="2 1000 0.75 0.75 2"关于contig的过滤

    has fewer than minReads (2) reads, or（这个值可以设置为5）
    is shorter than minLength (1000), or
    has a single read spanning singleReadSpan percent (75%) of the contig, or
    has less than lowCovDepth (2) coverage over at least lowCovSpan fraction (0.75) of the contig

对于低覆盖数据correctedErrorRate=0.075（4.5%-7.5%或者更多）也可以大于1%
对于高覆盖度数据correctedErrorRate=0.040（4.0%-4.5%），默认The default is 0.045 for PacBio reads，也可以小于1%

如果是AT（GC）富集的样本，建议设置corMaxEvidenceErate=0.15

三、Canu的简单使用

1.   设置spec文件

$ vi spec.txt

useGrid=1
gridOptions=-S /bin/bash -q all.q -l mem_free=60g
gridEngineThreadsOption=-pe mpi THREADS
gridEngineMemoryOption=-l mem_total=MEMORY
corOutCoverage=80
ovbMemory=8g
maxMemory=500g
maxThreads=48
ovsMemory=8-500g
ovsThreads=4
oeaThreads=4

2.   运行

$ canu -s spec.txt -p ecoli -d ecoli-auto genomeSize=4.8m -pacbio-raw p6.25x.fastq

========================================================
sudo apt install gnuplot
命令行绘图工具
实例1
#终端输入gnuplot进入命令行，运行脚本load "脚本路径及文件名"
#type gnuplot then type load 'sin_cos_tan.dem'
#设置标题及字体
set title "Simple Plots" font ",20"
#设置图例在箱子左边
set key left box
#设置样点数量，越多越平滑
set samples 100
#设置数据类型
set style data points

#绘制,plot 自变量 因变量（多个因变量之间用逗号，可实现多种图同时呈现）
plot [-10:10] sin(x),cos(x),tan(x)

实例2
#type gnuplot then type load 'sin_cos_tan.dem'
#设置标题及字体
set title "Simple Plots" font ",20"
#设置图例在箱子左边
set key left box
#设置样点数量，越多越平滑
set samples 100
#设置数据类型
set style data points

#绘制,plot 自变量 因变量（多个因变量之间用逗号，可实现多种图同时呈现）
plot [-pi/2:pi] cos(x),-(sin(x) > sin(x+1) ? sin(x) : sin(x+1))

实例3
 set title "3D surface from a grid (matrix) of Z values"
  2 set xrange [-0.5:4.5]
  3 set yrange [-0.5:4.5]
  4 
  5 set grid
  6 set hidden3d
  7 $grid << EOD
  8 5 4 3 1 0
  9 2 2 0 0 1
 10 0 0 0 1 0
 11 0 0 0 2 3
 12 0 1 2 4 3
 13 EOD
 14 splot '$grid' matrix with lines notitle



（1）基础

set title "Some math functions" // 图片标题

set xrange [-10:10] // 横坐标范围

set yrange [-2:2] // 纵坐标范围

set zeroaxis // 零坐标线

plot (x/4)*2, sin(x), 1/x // 画函数

 

（2）三维曲线

splot sin(x), tan(x) // 画sin(x)和tan(x)的三维曲线

 

（3）多条曲线

plot sin(x) title 'Sine', tan(x) title 'Tangent' // 画多条曲线，并且分别指定名称

 

从不同数据文件中提取信息来画图：

plot "fileA.dat" using 1:2 title 'data A', \

        "fileB.dat" using 1:3 title 'data B'

 

（4）点和线风格

plot "fileA.dat" using 1:2 title 'data A' with lines, \

        "fileB.dat" using 1:3 title 'data B' with linespoints

可以使用缩写：

using title and with can be abbreviated as u t and w.

 

plot sin(x) with line linetype 3 linewidth 2

或plot sin(x) w l lt 3 lw 2

plot sin(x) with point pointtype 3 pointsize 2

或plot sin(x) w p pt 3 ps 2

 

颜色列表大全：

http://www.uni-hamburg.de/Wiss/FB/15/Sustainability/schneider/gnuplot/colors.htm

用法：lc grb "greenyellow"

 

线条颜色，点的为pointtype

linetype 1 // 红色

linetype 2 // 绿色

linetype 3 // 蓝色

linetype 4 // 粉色

linetype 5 // 比较浅的蓝色

linetype 6 // 褐色

linetype 7 // 橘黄色

linetype 8 // 浅红色

 

线条粗细，点的为大小pointsize

linewidth 1 // 普通的线

linewidth 2 // 比较粗

linewidth 3 // 很粗

 

（5）常用

replot // 重绘

set autoscale // scale axes automatically

unset label // remove any previous labels

set xtic auto // set xtics automatically

set ytic auto // set ytics automatically

set xtics 2 // x轴每隔2设一个标点

set xlabel "x axe label"

set ylabel 'y axe label"

set label 'Yield Point" at 0.003,260 // 对一个点进行注释

set arrow from 0.0028,250 to 0.003,280 // 两点之间添加箭头

set grid // 添加网格

reset // gnuplot没有缩小，放大后只能reset后重绘，remove all customization

set key box // 曲线名称加框

set key top left // 改变曲线名称位置

set format xy "%3.2f" // x轴和y轴坐标格式，至少有3位，精确到小数点后两位

quit、q、exit // 退出程序

 

（6）绘制多个图

set size 1,1 // 总的大小

set origin 0,0 // 总的起点

set multiplot // 进入多图模式

set size 0.5,0.5 // 第一幅图大小

set origin 0,0.5 // 第一幅图起点

plot sin(x)

set size 0.5,0.5

set origin 0,0

plot 1/sin(x)

set size 0.5,0.5

set orgin 0.5,0.5

plot cos(x)

set size 0.5,0.5

set origin 0.5,0

plot 1/cos(x)

unset multiplot

 

（7）输出文件

set terminal png size 1024, 768 // 保存文件格式和大小

set output "file.png" // 保存文件名称

set terminal X11 // 重新输出到屏幕

 

（8）脚本

a.plt //后缀名为plt

gnuplot> load 'a.plt'

或gnuplot a.plt

save "graph.gp"

或save "graph.plt"

传入参数

call "a.plt" param1 param2

param1、param2在a.plt中对应于$0、$1

 

（9）字体设置

输出错误：

Could not find/open font when opening font "arial", using internal non-scalable font

下载RPM包：

wget http://www.my-guides.net/en/images/stories/fedora12/msttcore-fonts-2.0-3.noarch.rpm

rpm -ivh msttcore-fonts-2.0-3.noarch.rpm

设置环境变量：

修改/etc/profile或~/.bashrc，这样设置可以固定下来。

export GDFONTPATH="/usr/share/fonts/msttcore"

export GNUPLOT_DEFAULT_GDFONT="arial"

. /profile/etc

OK，现在可以使用arial字体了。
================================================================
sudo apt install octave
linux下像matlab一样的软件


sudo apt install qtiplot
Linux下科学绘图软件
====================================================
如果在终端不想复制的话可以用命令进入文件夹
cd `ls|sed -n '2p'`

``内是行内命令的输出作为新的命令的参数
这个表示进入ls后的第二个文件夹

==============================
fastaqc

sudo apt install gimp

sudo apt install gem
sudo gem install bio

conda install bioawk

conda install qiime
=========================================================
bow.sh
#$ -S /bin/sh
/hellogene/scgene01/bio/bin/bwa index P152L1_10.result
/hellogene/scgene01/bio/software/bowtie2-2.2.5/bowtie2-build P152L1_10.result long
/hellogene/scgene01/bio/software/bowtie2-2.2.5/bowtie2 -x long -1  ../P152L1_10/P152L1_10.R1.fq -2 ../P152L1_10/P152L1_10.R2.fq -p 25 | /hellogene/scgene01/bio/bin/samtools view -SbF 0x804 - | /hellogene/scgene01/bio/bin/samtools sort -@ 25 - > long.srt.bam 
/hellogene/scgene01/bio/bin/samtools index long.srt.bam



==========================================================
1.lib.conf文件
#maximal read length
max_rd_len=150
[LIB]
#average insert size
avg_ins=500
#if sequence needs to be reversed
reverse_seq=0
#in which part(s) the reads are used
asm_flags=3
#use only first 100 bps of each read
rd_len_cutoff=150
#in which order the reads are used while scaffolding
rank=1
# cutoff of pair number for a reliable connection (at least 3 for short insert size)
pair_num_cutoff=3
#minimum aligned length to contigs for a reliable read location (at least 32 for short insert size)
map_len=32
#a pair of fastq file, read 1 file should always be followed by read 2 file
q1=/hellogene/scgene03/prj/SCG-70616-198/01.fq/part_R1.fq
q2=/hellogene/scgene03/prj/SCG-70616-198/01.fq/part_R2.fq


2.kmer.sh
SOAPdenovo2-bin-LINUX-generic-r240/SOAPdenovo-127mer all -s lib.conf -o genome -K 96 -p 32

============================================
remap.sh
#$ -S /bin/sh
if [ $# -eq 0 ]; then
        echo "Program: Package the sequecing data and statics for mtFish sequencing"
        echo "Usage: $0 <genome.fasta> <fq1> <fq2>"
        exit 1
fi


ref=$1
fq1=$2
echo '#$ -S /bin/sh' > remap.sh
echo "/hellogene/scgene01/bio/bin/bwa mem -t 32 ${ref}  ${fq1} | /hellogene/scgene01/bio/bin/samtools view -SbF 0x804 - |/hellogene/scgene01/bio/bin/samtools sort -@ 32 - > genome.map.bam" >> remap.sh 
echo  '/hellogene/scgene01/bio/bin/samtools index genome.map.bam'  >> remap.sh
echo '/hellogene/scgene01/bio/bin/samtools depth genome.map.bam > map.dp' >>remap.sh
qsub -cwd -l vf=12g,p=32 -q prj.q remap.sh


remap.sh
#$ -S /bin/sh
/hellogene/scgene01/bio/bin/bwa mem -t 32 ../../all.cdna  ../../01.fq/DYM1_RRA126332-V_1.clean.fq.gz | /hellogene/scgene01/bio/bin/samtools view -SbF 0x804 - |/hellogene/scgene01/bio/bin/samtools sort -@ 32 - > genome.map.bam
/hellogene/scgene01/bio/bin/samtools index genome.map.bam
/hellogene/scgene01/bio/bin/samtools depth genome.map.bam > map.dp

单端测序：
bwa比对
bwa men -t 4 ref/all.cdna v1.fq.gz 产生sam文件
samtools view -SbF genome.map.sam (-S自动检测输入类型，-b输出bam格式，-F没有flag的reads）
samtools sort -@ 4 genome.map.bam(默认按照位置排序）
samtools index genome.map.bam (这里建立索引产生genome.map.bam.bai)
samtools depth genome.map.bam > map.dp (测序深度）
python averdep.py map.dp（平均深度）
-----------------------------------------------------------------
averdep.py
 # -*- coding: utf-8 -*- 
'''''''''''''''''''''''''''''''''''''''''''''
在多序列的fasta文件remap到fastq后，会产生一个map.dp
的文件，将其拷贝至一处，运行python averdep.py map.dp
会产生一个map.dp.out的文件，这个文件第一列是多序列的fasta
文件中的序列id，第二列是多序列的fasta文件的序列的平均深度
（注：在remap的时候序列描述行中如果有空格remap会只取第一个空格前的部分作为id）
Author: lurui@scgene.com
Date: 2016.4.20
'''''''''''''''''''''''''''''''''''''''''''''
from __future__ import division
import re
import sys
from Bio import SeqIO

def main(depfile):
    outfile = open(depfile+'.out', 'w')
    dep ={}
    dep1={}
    fa_len={}
    for i in SeqIO.parse(fa,'fasta'):
	fa_len[i.id]=len(i.seq)
    for i in open(depfile, 'r'):
	eachlst=i.strip().split()
        #eachlst = re.findall(r'\w+', i)
        seqid = eachlst[0]
        seqdep = int(eachlst[-1])
        dep.setdefault(seqid, []).append(seqdep)
    	dep1.setdefault(seqid,0)
	dep1[seqid]=dep1[seqid]+1
    newdep ={}
    for k, v in dep.items():
        averdep = average(v)
        newdep[k]=[averdep,round(float(dep1[k])/fa_len[k],2)]
    sortdep = sorted(newdep.iteritems(), key = lambda d:d[1], reverse=True)
    print >> outfile, '%s\t%s\t%s' %('Seqid', 'Average_depth','cov')
    for j in sortdep:
        print >> outfile, '%s\t%s\t%s' %(j[0], j[1][0],j[1][1])
    outfile.close()
    
def average(lst):
    num = len(lst)
    summary = sum(lst)
    aver = round(summary / num, 2)
    return aver
    
if __name__=='__main__':
    if len(sys.argv) == 3:
        scgene, depfile, fa= sys.argv
        main(depfile)
    else:
        print 'Usage: <python> <script> <depthfile>'
        exit(2)
---------------------------------------------------------------
range.py
#!/usr/bin/env python
import os
import re
fr=open('map.dp.out')
fw=open('dprange','w')
firstline=fr.readline()
line_fh=fr.readlines()
count=len(line_fh)
a=0
b=0
c=0
d=0
e=0
f=0
g=0
h=0
i=0
j=0
for eline in line_fh:
	eline=eline.replace('\t',' ')
	eline=eline.split()
	dp=eline[2]
	dp=float(dp)
	if dp>=0.0 and dp<0.1:
		a=a+1
	elif dp>=0.1 and dp<0.2:
		b=b+1
	elif dp>=0.2 and dp<0.3:
		c=c+1	
	elif dp>=0.3 and dp<0.4:
		d=d+1
	elif dp>=0.4 and dp<0.5:
		e=e+1
	elif dp>=0.5 and dp<0.6:
		f=f+1
	elif dp>=0.6 and dp<0.7:
		g=g+1
	elif dp>=0.7 and dp<0.8:
		h=h+1
	elif dp>=0.8 and dp<0.9:
		i=i+1
	elif dp>=0.9 and dp<=1.0:
		j=j+1
a=float(a)/count*100
b=float(b)/count*100
c=float(c)/count*100
d=float(d)/count*100
e=float(e)/count*100
f=float(f)/count*100
g=float(g)/count*100
h=float(h)/count*100
i=float(i)/count*100
j=float(j)/count*100
fw.write("dprange:\n0~0.1:\t%.2f\n"%a)
fw.write("0.1~0.2:\t%.2f\n"%b)
fw.write("0.2~0.3:\t%.2f\n"%c)
fw.write("0.3~0.4:\t%.2f\n"%d)
fw.write("0.4~0.5:\t%.2f\n"%e)
fw.write("0.5~0.6:\t%.2f\n"%f)
fw.write("0.6~0.7:\t%.2f\n"%g)
fw.write("0.7~0.8:\t%.2f\n"%h)
fw.write("0.8~0.9:\t%.2f\n"%i)
fw.write("0.9~1.0:\t%.2f\n"%j)
fw.write("total :%d"%count)

fr.close()
fw.close()
------------------------------------------------------------
python ../Bam2ReadNumber.py ref/all.cdna genome.map.bam map.dp
Bam2ReadNumber.py
#! /usr/bin/python

import os
import sys
from Bio import SeqIO
def main(cds,bam,bam_dip):
    cds_dic={}
    all_gene={}
    for i in SeqIO.parse(cds,'fasta'):
        cds_dic[i.id]=i.seq
        ID=i.id.split(".")[0]
        all_gene.setdefault(ID,0)
    gene_num=len(all_gene.keys())
    covRatioGene(bam,gene_num)
    PercentOfGeneCov(bam_dip,cds_dic)
def PercentOfGeneCov(bam_dip,cds_dic):
    bam_dip_dic={}
    result={}
    out=open("Percent_gene_body.xls",'w')
    out.write("#Percent_gene_body\tNumber\n")
    for i in open(bam_dip,'r'):
        line=i.strip().split()
        name='%s_%s' % (line[0],line[1])
        bam_dip_dic.setdefault(name,int(line[2]))
    for i,k in cds_dic.items():
        if len(k) < 1000:
            continue
        for x in range(1,101):
            num=0
            depth=0
            result.setdefault(x,0)
            percent=float(x)/100
            length=len(k)
            start=int(round(length*(percent-0.01)))+1
            end=int(round(length*percent))+1
            for y in range(start,end):
                name="%s_%s" % (i,y)
                bam_dip_dic.setdefault(name,0)
                depth+=bam_dip_dic[name]
                if bam_dip_dic[name] !=0:
                    num+=1
            if num != 0:
                result[x]+=depth/num
    for x in range(1,101):
        out.write('%s\t%s\n' % (x,result[x]))
    out.close()
def covRatioGene(bam,gene_num):
    bam_dic={}
    gene_list=[]
    out=open("#Satutation_distrebution.xls",'w')
    os.system("samtools view %s > genome.bam.txt" % (bam))
    out.write("#Size\tPercent(%)\n")
    for i in open('genome.bam.txt','r'):
        line=i.strip().split("\t")
        gene=line[2].split('.')[0]
        ID=line[0]
        bam_dic.setdefault(ID,0)
        if bam_dic[ID]==0:
            gene_list.append(gene)
            bam_dic[ID]+=1
    begin_num=0
    Reads=len(bam_dic.keys())
    for i in range(1,101):
        result={}
        percent=float(i)/100
        size=int(round(Reads*percent))
        for x in gene_list[0:size]:
            result.setdefault(x,0)
        result_gene_num=len(result.keys())
        out.write("%s\t%s\n" % (size,round(float(result_gene_num)/gene_num*100,2)))
    out.close()
    os.system("rm -f genome.bam.txt")
if len(sys.argv) ==4:
    main(sys.argv[1],sys.argv[2],sys.argv[3])
if len(sys.argv) !=4:
    print "Usage: %s <genome> <not sort bam> <depth.txt>" 
-------------------------------------------------------------------

1.对参考基因组建立比对的索引
hisat2-2.1.0/hisat2-build -p 32 DataBase/Special/Oryza_sativa/Japonica/msu7.0/all.con genome_hisat
2.hisat比对产生sam文件转为bam文件并排序
hisat2-2.1.0/hisat2 -p 32 --dta -5 20 -3 20 -x genome_hisat -1 ../../01.fq/TYM1_RRA126335-V_1.clean.fq.gz  -2 ../../01.fq/TYM1_RRA126335-V_2.clean.fq.gz  |/hellogene/scgene01/bio/bin/samtools view -SbF 0x804 - | /hellogene/scgene01/bio/bin/samtools sort -@ 25 - > hisat2.srt.bam
3.对参考注释文件输出gtf文件（-A gene abundance estimation output file
-B enable output of Ballgown table files which will be created in the same directory as the output GTF (requires -G, -o recommended)
-b enable output of Ballgown table files but these files will be     created under the directory path given as <dir_path>
-G <guide_gff>   reference annotation to include in the merging (GTF/GFF3)
-o <out_gtf>     output file name for the merged transcripts GTF
-l <label>       name prefix for output transcripts (default: MSTRG)
-T <min_tpm>     minimum input transcript TPM to include in the merge
-p number of threads (CPUs) to use (default: 1)

stringtie-1.3.3b.Linux_x86_64/stringtie -G /hellogene/scgene01/DataBase/Special/Oryza_sativa/Japonica/msu7.0/all.gff3 -l T2 -o genome.fa.gtf -e -p 8 -A genome.gene.xls -B hisat2.srt.bam

双端测序：
HISAT2提供两个Python脚本将GTF文件转换成hisat2-build能使用的文件：
extract_exons.py Sus_scrofa.Sscrofa11.1.90.chr.gtf > genome.exon
extract_splice_sites.py Sus_scrofa.Sscrofa11.1.90.chr.gtf > genome.ss
最后创建Index
hisat2-build –ss genome.ss –exon genome.exon Sus_scrofa.Sscrofa11.1.dna.toplevel.fa Sus_tran 
Hisat2比对
将RNA-seq的测序reads使用hisat2比对
hisat2 -p 8 –dta -x ./ref/Sus_tran/Sus_tran -1 ./fastq/Blast_1.clean.fq -2 ./fastq/Blast_2.clean.fq -S ./hisat2-out/Blast.sam 

HTSeq安装
使用pip直接下载：
pip install HTSeq
conda install HTSeq
htseq-count 计数
将sam文件转换为bam文件
samtools view -S ./hisat2-out/Blast.sam -b > ./BAM/Blast.bam
bam文件排序#因为是双端测序，必须对bam文件排序
samtools sort -n ./BAM/Blast.bam ./BAM/Blast_sort.bam
samtools view -h ./BAM/Blast_sort.bam > ./SAM/Blast_sort.sam
htseq-count -s no ./SAM/Blast_sort.sam genes.gtf > ./reads count/Blast.count

stringtie转录本处理

1、 stringtie组装转录本(首先将sam文件转换为bam文件，并排序；然后对每个样本进行转录本组装)

for ele in Blast ICM Morula Oocyte P1_cell P2_cell P4_cell P8_cell PFF TE
do
echo -e “samtools view -S ele.sam−b>
ele.bam\nsamtools sort -@ 8 ele.bamele.sorted\nstringtie -p 8 -G Sus.gtf -o ele.gtfele.sorted.bam” >> out.sh
done 

2 、stringtie合并转录本（将所有样本的转录本进行合并）
stringtie –merge -p 8 -G Sus.gtf -o stringtie_merged.gtf mergelist.txt #mergelist.txt是自己创建的

for ele in Blast ICM Morula Oocyte P1_cell P2_cell P4_cell P8_cell PFF TE
do
echo -e “./$ele.gtf” >> mergelist.txt
done 

3、stringtie评估表达量（计算表达量并且为Ballgown包提供输入文件）
for ele in Blast ICM Morula Oocyte P1_cell P2_cell P4_cell P8_cell PFF TE
do
echo -e “stringtie -p 8 -G stringtie_merged.gtf -e -B -o ballgown/ele/ele.gtf $ele.sorted.bam” >> out2.sh
done 

在-B 指定的文件夹下生成特定的文件
e2t.ctab e_data.ctab i2t.ctab i_data.ctab t_data.ctab
e即外显子、i即内含子、t转录本；e2t即外显子和转录本间的关系，i2t即内含子和转录本间的关系，t_data即转录本的数据 

Ballgown表达量分析

1、 Ballgown的安装
source(“http://bioconductor.org/biocLite.R“)
biocLite(“ballgown”)
2、文件准备与分析
将数据的分组信息写入一个csv文件，此处phenodata.csv文件 

3、运行R脚本，分析
Rscript expr.R

library(ballgown)
library(genefilter)
a <- read.csv(“pheno_data.csv”)
bg <- ballgown(dataDir = ‘ballgown’, samplePattern = “Sample”, pData = a)
bg_filt <- subset(bg, “rowVars(texpr(bg)) > 0.1”, genomesubset=TRUE)
gene_expression <- gexpr(bg_filt)
write.csv(gene_expression, “./FPKM/gene_expression.csv”)
transcripts_expression <- texpr(bg_filt)
write.csv(transcripts_expression, “./FPKM/transcripts_expression.csv”)


===============================================================
软件安装
下载StringTie并解压：
tar zxvf stringtie-1.1.2.Linux_x86_64.tar.gz
将HISAT2目录添加到环境变量：
vi ~/.bashrc
在文件末位添加：
export PATH=/home/chenwen/bin/stringtie-1.1.2.Linux_x86_64:$PATH
保存退出
source ~/.bashrc

StringTie的输入文件
HISAT的输出文件为SAM格式，需要经过两步转换成StringTie可以使用的BAM格式：
1. SAM转BAM，并排序：
samtools view -S -b input.sam | samtools sort – input.sorted
2.修改HI标签：
samtools view -h input.bam | perl -ne ‘if(/HI:i:(d+)/) {$m=$1-1; $_ =~ s/HI:i:(d+)/HI:i:$m/} print $_;’| samtools view -bS – > input.correct.bam

运行StringTie
stringtie SRR534294.correct.bam -p 16 -G genes.gtf -B -o stout/transcripts.gtf
-p 指定线程数，默认1
-G 指定参考的转录组注释文件
-B 生成用于Ballgown 分析的文件
-o 指定输出文件
更多参数请查看HISAT2的操作手册：
https://ccb.jhu.edu/software/stringtie/index.shtml?t=manual
===========================================================================
5.序列组装（assembly）
原始代码：
stringtie -p 16 -G /genes/brassica.gff（基因组注释文件） -o /data/transcripts/【sample name】.gtf（输出，比对结果，gtf文件） -l 【sample name】（命名规则）  /data/bamfiles/【sample name】.bam（输入，bam文件所在目录）
封装成BASH脚本：
#! /bin/bash
# stringtie 1st step
while read line
do
      stringtie -p 16 -G/genes/brassica.gff -o /data/transcripts/${line}.gtf -l ${line}/data/bamfiles/${line}.bam

done < /files/samples.txt
6.Merge
stringtie --merge -p 16 -G /genes/brassica.gff -o stringtie_merged.gtf/data/transcripts/mergelist.txt
gffcompare检测组装结果：
gffcompare -r /genes/brassica.gff -G -o merged stringtie_merged.gtf

7.计算表达量并输出成ballgown格式
原始代码：
stringtie -e -B -p 16 -G /data/transcripts/stringtie_merged.gtf（用merge生成的gtf文件代替基因组注释） -o /data/ballgown/$【sample name】/$【sample name】.gtf（输出为ballgown所需的输入格式） /data/bamfiles/【sample name】.bam（输入，bam文件）
封装成BASH脚本：
#! /bin/bash
# stringtie 3rd step
while read line
do
        stringtie -e -B -p 16 -G/data/transcripts/stringtie_merged.gtf -o /data/ballgown/${line}/${line}.gtf/data/bamfiles/${line}.bam
done < /files/samples.txt
8.ballgown分析差异表达基因（在R中进行）
>install.packages(‘devtools’)
>source('http://www.bioconductor.org/biocLite.R')
>biocLite(c(’ballgown’, 'genefilter’,'dplyr’,'devtools’))
>library(ballgown)
>library(genefilter)
>library(dplyr)
>library(devtools)
>file <- ballgown(dataDir='/data/ballgown',samplePattern=【sample名前缀】）输入基因表达数据
>samplesNames(file)查看ballgown文件中各样本的排列顺序
>pData(file) <- read.csv(‘/data/geuvadis_phenodata.csv’）（导入自制的表型数据）
>filefilt <-subset(file,'rowVars(texpr(file))>1',genomesubset=TRUE)（过滤掉表达差异较小的基因）
>diff_genes <- stattest(filefilt,feature='gene',covariate=【自变量】,adjustvars=【无关变量】,meas='FPKM')（统计差异表达的基因）
将差异表达基因按pval从小到大排序，并写入csv文件：
>diff_genes <- arrange(diff_genes,pval)
>write.csv(diff_genes,'/data/diff_genes',row.names=FALSE)
------------------------------------------------------------------
heatmap_original_remove_zero.txt
gene    DYM1    DYM2    DYM3    TYM1    TYM2    TYM3
LOC_Os06g05820  1.8738  2.0727  2.5135  2.1371  2.2637  2.2891
LOC_Os01g55490  8.9735  8.9807  9.4652  10.6575 9.5136  10.5006
LOC_Os09g36360  8.3987  8.5437  9.3577  12.1213 10.8168 10.4552
LOC_Os07g38010  4.1274  5.3498  5.1393  4.6478  4.2816  4.6023

pca.R
library(gmodels)
library(ggplot2)
##输入
inname = "heatmap_original_remove_zero.txt" #input file name
outname = "PCA.pdf" # out PCA figure name
group <- c("DYM1","DYM2","DYM3","TYM1","TYM2","TYM3") #define the color for points
##step1: 数据的读取和处理
expr <- read.table(inname, header=T, row.names=1) #read the expr data
data <- t(as.matrix(expr)) #transpose the data
##step2：PCA分析
#do PCA
data.pca <- fast.prcomp(data,retx=T,scale=T,center=T) #对数据进行z-score归一化
##step3： PCA结果解析
#fetch the proportion of PC1 and PC2
a <- summary(data.pca)
tmp <- a[4]$importance
pro1 <- as.numeric(sprintf("%.3f",tmp[2,1]))*100
pro2 <- as.numeric(sprintf("%.3f",tmp[2,2]))*100
#将成分矩阵转换为数据框
pc = as.data.frame(a$x)
#给pc的数据框添加名称列和分组列
pc$group = group
pc$names = rownames(pc)
write.table(pc,"pca.txt") #输出画图数据
##step 4: 绘图
#draw PCA plot figure
xlab=paste("PC1(",pro1,"%)",sep="")
ylab=paste("PC2(",pro2,"%)",sep="")
pca=ggplot(pc,aes(PC1,PC2)) +
geom_point(size=3,aes(color=group)) +
geom_text(aes(label=names),size=4,vjust=-1)+labs(x=xlab,y=ylab,title="PCA") + #vjust可调整点标签位置
geom_hline(yintercept=0,linetype=4,color="grey") +
geom_vline(xintercept=0,linetype=4,color="grey") +
theme_bw()
#保存结果
ggsave(outname,pca,width=12,height=8)Zeiformes	Zeidae Zeus faber 

----------------------------------------------
heatmap_original.txt
gene    DYM1    DYM2    DYM3    TYM1    TYM2    TYM3
LOC_Os06g05820  1.8738  2.0727  2.5135  2.1371  2.2637  2.2891
LOC_Os10g27460  0.0     0.0     0.0     0.0     0.0     0.0
LOC_Os02g35980  0.0     0.0     0.0     0.0     0.0     0.0
LOC_Os09g23260  0.0     0.0     0.0     0.0     0.0     0.0
LOC_Os01g41670  0.0     0.0     0.0     0.0     0.0     0.0
LOC_Os02g13480  0.0     0.0     0.0     0.0     0.0     0.0
LOC_Os01g55490  8.9735  8.9807  9.4652  10.6575 9.5136  10.5006

library("pheatmap")
pdf("heatmap.pdf",width = 10, height = 10)
data=read.table("heatmap_original_remove_zero.txt",header=T,row.names=1,sep="\t")
matrix=cor(data) #计算相关系数
write.table(matrix,"heatmap_matrix.txt",sep="\t")
pheatmap(matrix,cluster_rows=F,cluster_cols=F, display_numbers=T, cellwidth = 85, color=colorRampPalette(c("white", "steelblue"))(100)) #行和列都不聚类，并且在热图中显示数值
------------------------------------------------------
sudo apt install mrbayes
远距离物种用ML最大似然法
近距离物种用MB MrBayes
mb -i xxx.nexus

1.物种基因
2.muscle比对
3.产生.fsa文件
4.mega转为.nexus格式
5.mb -i xxx.nexus
======================================
spades.sh
#$ -S /bin/sh
/hellogene/scgene01/opt/bin/python /hellogene/scgene01/bio/pip/ass/SPAdes-3.5.0-Linux/bin/spades.py -k 79,97 --only-assembler -1 /hellogene/scgene03/prj/SCG-70814-202/01.fq/RK80420HM_DSW62808-V_1.clean.fq.gz -2 /hellogene/scgene03/prj/SCG-70814-202/01.fq/RK80420HM_DSW62808-V_2.clean.fq.gz -t 32 -m 100 -o 03.ass/spadeAss

bow.sh
#$ -S /bin/sh
/hellogene/scgene01/bio/bin/bwa index Agelas_sp._mitochondrion_complete_genome.fasta
/hellogene/scgene01/bio/bin/bwa mem -t 14 Agelas_sp._mitochondrion_complete_genome.fasta  ../01.fq/RK80420HM_DSW62808-V_1.clean.fq.gz ../01.fq/RK80420HM_DSW62808-V_2.clean.fq.gz | /hellogene/scgene01/bio/bin/samtools view -SbF 0x804 - | /hellogene/scgene01/bio/bin/samtools sort -@ 8 - > genome.map.bam
/hellogene/scgene01/bio/bin/samtools index genome.map.bam

Usage: bwa mem [options] <idxbase> <in1.fq> [in2.fq]
===========================================
sudo apt install staden线粒体组装
sudo apt install clustalx
sudo apt install clustalo

pip install khmer

conda install snpsift
conda install snpeff
conda install snpsplit

curl 'http://genome.ucsc.edu/cgi-bin/hgTables?hgsid=646311755_P0RcOBvAQnWZSzQz2fQfBiPPSBen&boolshad.hgta_printCustomTrackHeaders=0&hgta_ctName=tb_ncbiRefSeq&hgta_ctDesc=table+browser+query+on+ncbiRefSeq&hgta_ctVis=pack&hgta_ctUrl=&fbQual=whole&fbUpBases=200&fbExonBases=0&fbIntronBases=0&fbDownBases=200&hgta_doGetBed=get+BED' >mm10.refseq.bed

wget ftp://ftp.ccb.jhu.edu/pub/data/bowtie2_indexes/mm10.zip
wget ftp://ftp.ccb.jhu.edu/pub/data/bowtie2_indexes/hg19.zip

Reference genome

The reference genome should be a single whole FASTA file containg all chromosome data. This file shouldn't be compressed. For human data, typicall hg19/GRch37 or hg38/GRch38 assembly is used, which can be downloaded from following sites:

    hg19/GRch37: ftp://ftp.ncbi.nlm.nih.gov/sra/reports/Assembly/GRCh37-HG19_Broad_variant/Homo_sapiens_assembly19.fasta
    hg38/GRch38: http://hgdownload.cse.ucsc.edu/goldenPath/hg38/bigZips/hg38.fa.gz Remember to decompress hg38.fa.gz since it is gzipped and is not supported currently.



sudo apt install spark
sudo apt install scala
sudo apt install ocaml


tophat-cufflinks流程
http://www.360doc.com/content/17/0802/19/45954458_676165658.shtml

gff注释文件下载
https://www.gencodegenes.org/releases/19.html

for ((i=56;i<=58;i++));do hisat2 -t -x /mnt/f/rna_seq/data/reference/index/hg19/genome -1 /mnt/f/rna_seq/data/SRR35899${i}.sra_1.fastq.gz -2 /mnt/f/rna_seq/data/SRR35899${i}.sra_2.fastq.gz -S SRR35899${i}.sam ;done 
# 小鼠比对 
for ((i=59;i<=62;i++));do hisat2 -t -x /mnt/f/rna_seq/data/reference/index/mm10/genome -1 /mnt/f/rna_seq/data/SRR35899${i}.sra_1.fastq.gz -2 /mnt/f/rna_seq/data/SRR35899${i}.sra_2.fastq.gz -S SRR35899${i}.sam; done

================================================================
我们需要纠结是否真的需要比对，如果你只需要知道已知基因的表达情况，那么可以选择alignment-free工具（例如salmon, sailfish)，如果你需要找到noval isoforms，那么就需要alignment-based工具（如HISAT2, STAR）。到了这一篇的基因（转录本）定量，需要考虑的因素就更加多了，以至于我不知道如何说清才能理清逻辑。
定量分为三个水平

    基因水平(gene-level)
    转录本水平(transcript-level)
    外显子使用水平(exon-usage-level)。
    在基因水平上，常用的软件为HTSeq-count，featureCounts，BEDTools, Qualimap, Rsubread, GenomicRanges等。以常用的HTSeq-count为例，这些工具要解决的问题就是根据read和基因位置的overlap判断这个read到底是谁家的孩子。值得注意的是不同工具对multimapping reads处理方式也是不同的，例如HTSeq-count就直接当它们不存在。而Qualimpa则是一人一份，平均分配。
    对每个基因计数之后得到的count matrix再后续的分析中，要注意标准化的问题。如果你要比较同一个样本(within-sample)不同基因之间的表达情况，你就需要考虑到转录本长度，因为转录本越长，那么检测的片段也会更多，直接比较等于让小孩和大人进行赛跑。如果你是比较不同样本（across sample）同一个基因的表达情况，虽然不必在意转录本长度，但是你要考虑到测序深度（sequence depth)，毕竟测序深度越高，检测到的概率越大。除了这两个因素外，你还需要考虑GC%所导致的偏差，以及测序仪器的系统偏差。目前对read count标准化的算法有RPKM（SE）, FPKM（PE），TPM, TMM等，不同算法之间的差异与换算方法已经有文章进行整理和吐槽了。但是，有一些下游分析的软件会要求是输入的count matrix是原始数据，未经标准化，比如说DESeq2，这个时候你需要注意你上一步所用软件会不会进行标准化。
    在转录本水平上，一般常用工具为Cufflinks和它的继任者StringTie， eXpress。这些软件要处理的难题就时转录本亚型（isoforms）之间通常是有重叠的，当二代测序读长低于转录本长度时，如何进行区分？这些工具大多采用的都是expectation maximization（EM）。好在我们有三代测序。
    上述软件都是alignment-based，目前许多alignment-free软件，如kallisto, silfish, salmon，能够省去比对这一步，直接得到read count，在运行效率上更高。不过最近一篇文献[1]指出这类方法在估计丰度时存在样本特异性和读长偏差。
    在外显子使用水平上，其实和基因水平的统计类似。但是值得注意的是为了更好的计数，我们需要提供无重叠的外显子区域的gtf文件[2]。用于分析差异外显子使用的DEXSeq提供了一个Python脚本（dexseq_prepare_annotation.py）执行这个任务。

小结

计数分为三个水平： gene-level, transcript-level, exon-usage-level
标准化方法： FPKM RPKM TMM TPM
==================================================================
STAR的安装

cd biosoft && mkdir STAR && cd STAR
wget https://github.com/alexdobin/STAR/archive/2.5.3a.tar.gz
tar -xzf 2.5.3a.tar.gz
cd STAR-2.5.3a

# for easy use, add bin/ to your PATH

下载需要参考基因组并进行index构建

# downloading dna index fasta file
nohup wget -r -np -nH -nd -R index.html -L ftp://ftp.ensembl.org/pub/release-90/fasta/homo_sapiens/dna_index/ &

# download gft annotation file
nohup wget ftp://ftp.ensembl.org/pub/release-90/gtf/homo_sapiens/Homo_sapiens.GRCh38.90.chr_patch_hapl_scaff.gtf.gz &

mkdir STAR_index && cd STAR_index
STAR --runMode genomeGenerate --genomeDir ~/reference/STAR_index/ --genomeFastaFiles ~/reference/genome/hg38/Homo_sapiens.GRCh38.dna.toplevel.fa --sjdbGTFfile ~/reference/genome/hg38/Homo_sapiens.GRCh38.90.chr_patch_hapl_scaff.gtf --sjdbOverhang 199

# --sjdbOverhang 数值为reads长度-1
# Mode 为generate
# --genomeFastaFiles --sjdbGTFfile 分别对应fasta文件和GTF文件

STAR的使用

# STAR的manual里面给了最基本的比对参数示例
STAR
--runThreadN NumberOfThreads
--genomeDir /path/to/genomeDir
--readFilesIn /path/to/read1 [/path/to/read2 ]

# 基本示例，
针对fastq.gz文件增加--readFilesCommand gunzip -c 参数/--readFilesCommand zcat参数，针对bzip2文件使用--readFilesCommand bunzip2 -c参数
STAR --runThreadN 20 --genomeDir ~/reference/STAR_index/ --readFilesCommand zcat --readFilesIn ~/RNA-seq/LiuPing_data/RNA-seq/SC_w2q20m35_N_1.fq.gz ~/RNA-seq/LiuPing_data/RNA-seq/SC_w2q20m35_N_2.fq.gz

# 输出unsorted or sorted bam file
--outSAMtype BAM Unsorted 实际上就是-name 的sort，下游可以直接接HTSeq
--outSAMtype BAM SortedByCoordinate
--outSAMtype BAM Unsorted SortedByCoordinate 两者都输出

额外参数说明

# 单独指定注释文件，而不用在构建的时候使用
--sjdbGTFfile /path/to/ann.gtf
--sjdbFileChrStartEnd /path/to/sj.tab

# ENCODE参数

# 减少伪junction的几率
--outFilterType BySJout

# 最多允许一个reads被匹配到多少个地方
--outFilterMultimapNmax 20

# 在未有注释的junction区域，最低允许突出多少个bp的单链序列
--alignSJoverhangMin 8

# 在有注释的junction区域，最低允许突出多少个bp的单链序列
--alignSJDBoverhangMin 1

# 过滤掉每个paired read mismatch数目超过N的数据，999代表着忽略这个过滤
--outFilterMismatchNmax 999

# 相对paired read长度可以允许的mismatch数目，如果read长度为100，数值设定为0.04，则会过滤掉100*2*0.04=8个以上的数据
--outFilterMismatchNoverReadLmax 0.04

# 最小的intro长度
--alignIntronMin 20

# 最大的intro长度
--alignIntronMax 1000000

# maximum genomic distance between mates，翻译不出来，自行理解
--alignMatesGapMax 1000000

STAR的输出

    STAR可以根据你的参数设定输出多个结果文件，包含各种信息，下面对默认参数情况下的输出文件做了一个详细的展示，有些不好翻译的地方我选择使用原汁原味的manual text

    Aligned.out.sam
    Aligned.out.sam当然就是我们的比对结果啦！

E00516:168:H37WKCCXY:8:1101:6400:59130    99    1    92836373    255    20M1063N129M    =    92837548    4244    GGCTTGTCTATCCCTCACAGTACCAAACGATTCCCTGGTTATGATTCTGAAAGCAAGGAATTTAATGCAGAAGTACATCGGAAGCACATCATGGGCCAGAATGTTGCAGATTACATGCGCTACTTAATGGAAGAAGATGAAGATGCTTA    AAFFFJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJJ    NH:i:1    HI:i:1    AS:i:289    nM:i:0

# 我截取了一条比对信息
我们来看一下最后面的 NH:i:1  HI:i:1  AS:i:289    nM:i:0
NH:i:后面的数值代表着此条read比对到几个loci，1代表着unique map，数值大于1代表着multi-mappers
HI:i:后面的数值attrbiutes enumerates multiple
alignments of a read starting with 1，下游分析接cufflinks or stringtie的时候需要使用参数--outSAMattrIHstart 0
AS:i:的数值代表着local alignment score (paired for paired-edn reads)
nM:i:的数值代表着the number of mismatches per (paired) alignment, not to be confused with NM, which is the number of mismatches in each mate
关于下游处理工具的兼容性还需要使用者自己仔细参考manual

    Log.out文件
    Log.out文件记录了程序运行时的信息，可以用来回溯错误信息。

========================================================
conda install STAR
应用时用STAR
conda install express

    Subread: a general-purpose read aligner which can align both genomic DNA-seq and RNA-seq reads. It can also be used to discover genomic mutations including short indels and structural variants.
    Subjunc: a read aligner developed for aligning RNA-seq reads and for the detection of exon-exon junctions. Gene fusion events can be detected as well.
    featureCounts: a software program developed for counting reads to genomic features such as genes, exons, promoters and genomic bins.
    Sublong: a long-read aligner that is designed based on seed-and-vote.
    exactSNP: a SNP caller that discovers SNPs by testing signals against local background noises.

conda install subread
下载安装subjunc,featurecounts
conda install bedtools

参考基因，注释等序列下载
https://ccb.jhu.edu/software/tophat/igenomes.shtml


