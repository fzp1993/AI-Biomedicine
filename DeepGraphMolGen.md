# DeepGraphMolGen, a multi-objective, computational strategy for generating molecules with desirable properties: a graph convolution and reinforcement learning approach

## 作者
Yash Khemchandani，Stephen O’Hagan，Soumitra Samanta，Neil Swainston，Timothy J. Roberts，Danushka Bollegala，Douglas B. Kell

## 刊物
Journal of Cheminformatics，2020

## 解决问题
* 交互绑定模型（interaction binding model）是利用图卷积网络从绑定数据（binding data）中学习的，实验获得的属性分数被认为具有潜在的重大误差（gross error）
* 生成分子的基于序列的表征模型必须学习输入数据的固有规则，但是他们用的输入是SMILES字符串是上下文敏感的表示，也就是添加一个额外的原子或括号可以极大地改变编码分子的“全局”结构，而不仅仅是“局部”的改变。但是使用图结构来把分子表示为图形的方法更有效，但是这些方法只被用于比较确定性性质的模型

## 创新点
* 生成具有期望相互作用性质（desired interaction properties）的新分子的问题作为一个多目标优化问题来解决
* 我们针对gross error问题，对模型采用了一个稳健的损失（robust loss）。对术语的组合（药物相似性和合成可及性）使用基于图卷积策略方法的强化学习进行优化
* 我们成功地将我们的方法扩展到使用多目标奖赏函数，在这种情况下，我们可以生成与多巴胺转运体结合的新分子，但不与去甲肾上腺素结合的新分子

## 思路概述
* 系统分为性质预测和分子生成两部分，使用图表示分子
* 性质预测部分是一个图卷积网络和一个前馈网络构成，使用Adaptive Robust Lose Function算loss
* 分子生成部分使用的是强化学习，奖励是分子约束（molecular constraint）和期望属性（desired properties），使用了图卷积策略网络（graph convolution policy networks）。这一部分是来指导专家预训练和对抗损失残生有效分子
### 性质预测
#### 特征表示
* 分子表示为有向图，一个原子表示为一个133维的向量，每个原子都有一个连接它的键，表示为14维向量，因此一个初始原子键特征向量表示为一个133+14维的向量，一共有Nbond个向量，也就是键的个数个向量。每个维度表示一个性质
  ![分子表示的维度含义](/pic/分子表示的维度含义.png "分子表示的维度含义")
* 初始原子键特征向量包含了重要的分子信息，GCN编码器可以合并到后面的层中
* 步骤：初始原子键特征向量经过线性层和一个RelU激活函数，得到0深度的信息向量，然后一个GCN将相邻键信息求和，然后重复这个步骤到N-1层深度，最后是一个Dense层、ReLU和Dropout，得到分子图嵌入表示
  ![特征表示图](/pic/特征表示图.png "特征表示图")

#### 回归
* 做性质预测，将表示通过全连接层，然后最后出一个预测分数，预测的是Ki值
* 损失函数有说法，说是Ki值的真实值是实验得到的，有实验误差，也就是训练集中有异常值，用一般的损失函数训练的话，泛化性不好。因此用robust loss来训练，包括pseudo-huber loss、cauchy loss等，但是每种损失函数都有超参数需要手动调，费时间。所以用了一个general robust loss，也就是用一个shape parameter $\alpha$ 来控制robustness（根据文中的意思就是 $\alpha$ 取不同值就是不同的robust loss）和一个scale parameter c来把loss的二次碗（quadratic bowl）控制在x=0的附近。然后文中把这两个超参也作为梯度训练的参数了
  ![回归图](/pic/回归图.png "回归图")
  ![loss函数](/pic/loss函数.png "loss函数")

### 分子生成
#### 分子表示
* 使用GCPN模型来生成分子，这与性质预测里面的特征表示是不一样的，表示为（A,E,F），其中A是邻接矩阵（n* n），F是原子特征矩阵（n* d），E是边张量（一条边有三种类型，所以是3* n* n的元素为0或1的张量）。

#### 强化学习
* 初始是从一个碳原子或其他一个分子，每一步添加一个键
* action是一个四维向量（第一个原子，第二个原子，键类型，终止符）
* 每一步的state是当前分子图
* reward包括intermediate reward和final reward。intermediate reward的含义是通过stepwise validity check就加一个小定值。final reward包括训练模型预测的pki、validity reward（是否有空间应变和是否违反锌官能团）、药物相似性QED的定量估计和合成可及性SA的评分。
* 还有adversarial reward，目的是使生成的分子类似于给定的一组分子，它设置为GAN的loss，文中表示为
  ![文中GAN](/pic/文中GAN.png "文中GAN")
  这里面的D是一个判别器网络，包括GCN和FFN，用来输出生成的分子是真是假。
* 在GCN中，是一个深度为L的网络，第l+1层的节点嵌入表示为
  ![GCN分子生成](/pic/GCN分子生成.png "GCN分子生成")
  Ei是第i个切片（边不是有三个类型吗，这就是第i个类型）加上单位阵。Di是Ei在第三个维度上的求和（就是一个求均值，可以理解为融合不同节点特征）。然后DiEiDi计算成一个权值给上一步的节点嵌入表示Hl，Wi是训练参数。Hi是一个（n+c）*d的矩阵，n是当前分子中节点个数（也就是说，下一步迭代这个n就有可能是n+1，因为加入了新的原子，但如果没有加入新原子，而是和老原子之间成键，那还是n），c是可能加入的原子类型个数（C,N,O），经过AGG生成节点表示，这个过程循环L次。
* 然后每一步的节点嵌入输入给四个MLP，第一个MLP是为了基于当前分子找到下一步需要成键的原子对中的第一个原子；第二个MLP是为了基于当前分子找到第二个原子（可以是老原子，也可以是新原子CNO）；第三个MLP是为了找到这个原子对的键的类型；第四个MLP是为了看是否结束生成过程。
  ![第一个MLP](/pic/第一个MLP.png "第一个MLP")
  ![第二个MLP](/pic/第二个MLP.png "第二个MLP")
  ![第三个和第四个MLP](/pic/第三个和第四个MLP.png "第三个和第四个MLP")
* 使用标准的PPO来训练策略梯度，让我不明白的是V怎么算的，是不是前面说的那几个reward，很简单的那种相加。同时还加了一个专家系统
  ![ppo](/pic/ppo.png "ppo")
  ![专家系统](/pic/专家系统.png "专家系统")

## 数据集
* 多巴胺转运体结合数据：www.bindingdb.org（https://bit.ly/2YACT5u） ，训练数据中一部分分子标记了Ki值，一部分分子标记了IC50值
* BindingDB数据集，用来评估强化学习模块，选取与生成的分子相似的分子结构
* ZINC数据集，用于强化学习中的专家系统

## 评价指标
* 性质预测中Hyperparameter optimization使用RMS
* TS

## 结论和实验结果
* 性质预测目标是生成小分子与多巴胺转运体相互作用，但不（尽可能）与去甲肾上腺素转运体相互作用
* 评估强化学习时考虑的是一个单一目标（多巴胺转运体相互作用）的分子生成，取出生成的前十个分子，从BindingDB中找到相似的分子，计算他们RDKit分子指纹之间的Tanimoto Similarity。生成的十个分子还预测了QED和SA
* 评估强化学习时还考虑多目标，与多巴胺转运体有高结合但与去甲肾上腺素转运体结合低得多的分子

## 存在问题