# ReSTIR

我们将介绍一种适用于动态场景光线追踪的方法来采样大量光源的一次弹射直接光照。该方法基于2005年的重采样重要性采样（RIS）技术，该技术从一种分布中提取一组样本，并使用另一种更接近被积函数的分布来从中选取一个加权子集。本文的不同之处在于，我们使用一个小规模的固定尺寸的“蓄水池”数据结构来存储被接受的样本，并使用关联采样算法（常被用于非图形学的应用中）来达到稳定的实时表现。

蓄水池的数据结构很简单，是一个固定尺寸的数组，但它通过复用时空邻域的统计信息来随机地、渐进地、分层地改进每个像素地直接光照采样的PDF。与实时降噪算法复用时空邻域的像素颜色不同，我们复用了**采样概率**，这也使得我们能得到无偏的结果。我们也给出了一种有偏的方法来进一步降低噪声，代价是在几何不连续的位置附近会更加黯淡。

### 原理

先给出$y$点反射的辐射度表示：

$L(y,\omega)=\int_{A}\rho(y,\overrightarrow{yx}\leftrightarrow\overrightarrow{\omega})L_e(x\to y)G(x\leftrightarrow y)V(x\leftrightarrow y)dA_x$

其中，$A$表示所有的光源表面，$\rho$表示BSDF，$V$表示相互可见性，G是含有距离平方反比和余弦加权的几何项。

简记形式为：$L=\int_Af(x)dx\quad where\space f(x)\equiv\rho(x)L_e(x)G(x)V(x)$

**1. 重要性采样**

一般表述形式为：

${\left<L\right>}_{is}^N=\frac{1}{N}\underset{i=1}{\overset{N}{\sum}}\frac{f(x_i)}{p(x_i)}\approx L$

其中，$N$为采样数，$x_i$为采样位置，$p(x_i)$为概率密度函数（PDF）。如果在任何$f(x)$不为零的位置，$p(x)$也都为正，则该重要性采样便是无偏的。理想的概率密度函数应当与原函数$f(x)$相关以减少方差。

实际上，直接按原函数比例取样是不现实的，因为会受可见性$V(x)$的影响。但我们仍然可以按被积函数中的部分项（BSDF项和自发光项）的比例取样。

**2. 多重重要性采样**

给定$M$个候选采样策略$p_s$，则多重重要性采样对每种采样策略分别选取$N_s$个样本，并最终得到一个加权估计：

${\left<L\right>}_{mis}^{M,N}=\underset{s=1}{\overset{M}{\sum}}\frac{1}{N_s}\underset{i=1}{\overset{N_s}{\sum}}w_s(x_i)\frac{f(x_i)}{p_s(x_i)}$

其中，权重函数满足归一化$\sum_{s=1}^{M}w_s(x)=1$，因而MIS仍然是无偏估计。

一般使用启发式的权重函数：$w_s(x)=\frac{N_sp_s(x)}{\sum_jN_jp_j(x)}$

**3. 重采样重要性采样（RIS）**

另一种可选的MIS采样策略则是按部分项乘积的近似比例采样。重采样重要性采样（RIS）就是这么做的，它从次优的原始分布$p$中生成$M\geq1$个候选样本$\mathbf{x}=\left\{x_1,...,x_M\right\}$，然后根据离散概率随机地从中选取一个序号$z\in\left\{1,...,M\right\}$：

$p(z|\mathbf{x})=\frac{\textrm{w}(x_z)}{\sum_{i=1}^M\textrm{w}(x_i)}\quad with\space \textrm{w}(x)=\frac{\hat{p}(x)}{p(x)}$

目标概率密度函数$\hat{p}(x)$可能是不存在实际的采样算法的（比如，$\hat{p}\propto\rho\cdot L_e\cdot G$），则单样本的RIS估计就可以表述为：

${\left<L\right>}_{ris}^{1,M}=\frac{f(y)}{\hat{p}(y)}\cdot\left(\frac{1}{M}\underset{j=1}{\overset{M}{\sum}}\textrm{w}(x_j)\right)$

其中，$y\equiv x_z$

直观上，RIS估计使用圆括号中的项作为系数来修正采样，使得$y$就好像是按$\hat{p}$采样的。当$M=1$是，RIS估计等同于IS。

多次重复RIS并平均结果，就可以得到N样本的RIS估计：

${\left<L\right>}_{ris}^{N,M}=\frac{1}{N}\underset{i=1}{\overset{N}{\sum}}\left(\frac{f(y_i)}{\hat{p}(y_i)}\cdot\left(\frac{1}{M}\underset{j=1}{\overset{M}{\sum}}\textrm{w}(x_{ij})\right)\right)$

只要$p,\hat{p}$在$f$非零处都为正值，则RIS也是无偏估计。简便起见，后续将按$N=1$使用。

一般地，图片中的每个像素$q$都有唯一的被积函数$f_q$以及相应的PDF $\hat{p}_q$

也可以混合使用MIS和RIS，即使用MIS生成初始PDF，再通过RIS优化。但这种方式会使得计算开销随MIS的增加而显著增加（平方倍），我们稍后使用一种全新的MIS方法来解决该问题。

**4. 加权蓄水池采样（WRS）**

加权蓄水池采样是一族从样本流中随机采样$N$个样本的算法。每一个流元素都有一个相应的权值$\textrm{w}(x_i)$，流元素$x_i$被选中的概率应为：

$P_i=\frac{\textrm{w}(x_i)}{\sum_{j=1}^{M}\textrm{w}(x_j)}$

蓄水池采样对每个流元素只处理一次，并且只在蓄水池中存储$N$个元素。蓄水池采样算法虽然分为元素$x_i$可以重复出现的替换型和不可重复出现的非替换型，但我们只需要考虑简单的替换型即可。

（$N=1$）蓄水池采样按顺序处理样本流，当处理完$m$个元素时，第$i$个流元素被选入蓄水池的概率为$\textrm{w}(x_i)/\sum_{j=1}^m\textrm{w}(x_j)$；对于之后处理的第$m+1$个流元素，则按概率$\frac{\textrm{w}_{m+1}}{\sum_{j=1}^{m+1}\textrm{w}_j}$选取，并随机替换掉蓄水池中的第$i$个样本，则样本$x_i$被保留的概率为：

$\frac{\textrm{w}(x_i)}{\sum_{j=1}^m\textrm{w}(x_j)}\left(1-\frac{\textrm{w}(x_{m+1})}{\sum_{j=1}^{m+1}\textrm{w}(x_j)}\right)=\frac{\textrm{w}(x_i)}{\sum_{j=1}^{m+1}\textrm{w}(x_j)}$

### 时空重用的流式RIS

接下来就是通过WRS来进行时空重采样以获得大量的可用样本。

但，原生的时空重采样方法是有偏的，因为不同的像素是按照不同的BRDF和表面方向采样的。有偏会导致几何不连续处的能量损失。

**使用蓄水池采样的流式RIS**

可以直接通过WRS将RIS转为流式算法。经验证，流式RIS已经比当前最好的动态光源BVH算法的效果更好，且流式RIS不依赖于预处理或复杂的数据结构，虽然高质量的渲染结果仍然需要更大的$M$值（候选样本数量），但内存开销固定为$M$个样本，计算开销随$M$线性增加。

**时空复用**

我们会发现，邻域像素的PDF通常具有相似性，例如邻域像素可能具有相似的BSDF。一种基本的复用方案是生成并存储逐像素的候选样本及其权值，然后在第二个pass中结合像素及其邻域像素来复用计算结果。由于权值计算已经在第一个pass中进行，所以第二个pass中复用邻域像素的计算结果开销更小。（该方案和2002年Bekaert等人的复用方案很相似，只不过他们是根据可见性复用样本）

不幸的是，上述方案不具有可行性，因为它需要缓存每一个候选样本。但是，我们可以通过蓄水池采样的一个重要特性来规避存储需求，它允许我们在不获取样本流的情形下混合多个蓄水池。

每个蓄水池的状态都包含当前选中的样本$y$和目前处理的所有候选样本的合计权值$\textrm{w}_{sum}$。为了混合两个蓄水池，我们可以将每个蓄水池的状态都当作是一个具有权值$\textrm{w}_{sum}$的新样本$y$，然后将它们送入一个新的蓄水池中，得到的结果在数学上是等价的。该方案使得我们只需要常数的处理时间，并且不必存储各个像素的样本流，只需要访问每个像素蓄水池的当前状态即可。考虑到邻域像素的目标分布$\hat{p}_{q'}$是不相同的，我们可以通过因子$\hat{p}_q(r.y)/\hat{p}_{q'}(r.y)$来修正过采样和欠采样的邻域像素。

现在，我们具有了空间范围的像素复用方案了。我们首先通过RIS为每个像素都生成$M$个候选样本，并将蓄水池结果存储在一张与图像尺寸相同的缓存中。第二步，每个像素都选取邻域的$k$的像素，并将它们的蓄水池和自身的蓄水池混合。至此，每个像素的时间开销为$O(k+M)$，但每个像素都拥有了$k\cdot M$个候选样本。

我们还可以继续迭代复用之前的结果，将上一个复用pass的结果作为输入，再次重复混合操作。迭代$n$次的算法开销就是$O(nk+M)$，而每个像素却能获得$k^nM$个候选样本（前提是每次迭代都使用不同的邻域样本）。当然，迭代复用的收益并不能无限提高，迭代几次用完所有可用的邻域像素之后，图像质量就趋于稳定了。

接下来我们讨论时间范围的像素复用方案。我们知道渲染出来的图像通常不是孤立的，而是有一个动画序列，因此前序图像也可以提供可供复用的候选样本。我们在每一帧的最后缓存下它们的蓄水池状态，留给下一帧复用。重复这样的操作，实际上是混合了所有前序帧的蓄水池，这可以大大提升图像质量。

最后，我们再讨论一下可见性的复用。我们需要认识到的是，即使候选样本的数量无限多，RIS也无法得到无噪声的渲染结果，也就是说，即使无限接近目标PDF$\hat{p}$，也无法完美地采样被积函数$f$。这是由于$\hat{p}$在一般情况下只能是描述非阴影路径的光照贡献，当候选样本量越来越多时，可见性问题引起的噪声开始逐步占据主导位置。不幸的是，越复杂的场景，可见性问题引起的噪声会越严重，为了解决该问题，我们将进行可见性的复用。

在进行时空复用之前，我们先计算一下所选样本$y$的可见性，如果$y$被遮挡，则抛弃该蓄水池，这也意味着被遮挡的样本不再向邻域传递。

### 无偏方法

我们先从对RIS开始，重新整理RIS的表述形式，可以得到：

$\left<L\right>_{ris}^{1,M}=f(y)\cdot\left(\frac{1}{\hat{p}(y)}\frac{1}{M}\underset{j=1}{\overset{M}{\sum}}\textrm{w}(x_j)\right)=f(y)W(\mathbf{x},z)$

则，$W$是样本$y\equiv x_z$的权值。

回顾蒙特卡洛估计的基本形式$f(y)/p(y)$，使用$W(\mathbf{x},z)$作为权值就意味着我们期望$W(\mathbf{x},z)=1/p(y)$。但$W(\mathbf{x},z)$是一个随机变量，我们无法确定输出的样本$y$来自于怎样的$\left\{\mathbf{x},z\right\}$。为了最终得到无偏的RIS，我们需要确保$W(\mathbf{x},z)=1/p(y)$。

回到之前的使用场景，我们知道抽取的样本实际上可能来自不同的PDF $p_i(x_i)$，那么，它们的联合PDF为：

$p(\mathbf{x})=\underset{i=1}{\overset{M}{\prod}}p_i(x_i)$

接下来，我们根据联合PDF离散地选取某个样本序号$z\in\left\{1,...,M\right\}$：

$p(z|\mathbf{x})=\frac{\textrm{w}_z(x_z)}{\sum_{i=1}^M\textrm{w}_i(x_i)}$

其中，$\textrm{w}_i(x)=\frac{\hat{p}(x)}{p_i(x)}$

于是有：

$p(\mathbf{x},z)=p(\mathbf{x})p(z|\mathbf{x})=\left[\underset{i=1}{\overset{M}{\prod}}p_i(x_i)\right]\frac{\textrm{w}_z(x_z)}{\sum_{i=1}^M\textrm{w}_i(x_i)}$

但样本集$\mathbf{x}$中，哪一个是$y$，我们并不能确定，既有可能是$x_1=y,z=1$，也可能是$x_2=y,z=2$。不过我们知道一些前提条件，从而确定一个集合范围：

$Z(y)=\left\{i|1\leq i\leq M\wedge p_i(y)>0\right\}$

为了得到关于$y$的PDF，我们可以直接计算边缘概率密度：

$p(y)=\underset{i\in Z(y)}{\sum}\underset{M-1\space times}{\underbrace{\int\cdots\int}}p(\mathbf{x}^{i\to y},i)\underset{M-1\space times}{\underbrace{dx_1\cdots dx_M}}$

其中，$\mathbf{x}^{i\to y}=\left\{x_1,...,x_{i-1},y,x_{i+1},...,x_M\right\}$是将第$i$个候选样本固定为$y$的简记形式。边缘概率密度的计算只对剩下的$M-1$个候选样本（对应的概率密度）积分。

于是，我们可以尝试验证RIS权值$W(\mathbf{x},z)$的期望。

我们先通过一个条件期望来表示$x_z=y$时权值的期望：

$\underset{x_z=y}{\mathbb{E}}[W(\mathbf{x},z)]=\underset{i\in Z(y)}{\sum}\frac{\int\cdots\int W(\mathbf{x}^{i\to y},i)p(\mathbf{x}^{i\to y},i)dx_1\cdots dx_M}{p(y)}$

上式经过化简可以得到：

$\underset{x_z=y}{\mathbb{E}}[W(\mathbf{x},z)]=\frac{1}{p(y)}\frac{|Z(y)|}{M}$

从而确定，当各个候选样本的PDF都在目标函数非零处不为零，且$|Z(y)|=M$时，RIS估计无偏。换句话说，如果存在某个候选样本的PDF在被积函数的某一段上为零，则$\frac{|Z(y)|}{M}<1$，进而导致RIS估计低于实际积分结果，在渲染结果上体现为“变暗”。

消除偏差的方法是对权值和使用特别指定的修正权重：

$W(\mathbf{x},z)=\frac{1}{\hat{p}(x_z)}\left[m(x_z)\underset{i=1}{\overset{M}{\sum}}\textrm{w}_i(x_i)\right]$

回到权值的期望计算，我们重新表述为：

$\underset{x_z=y}{\mathbb{E}}[W(\mathbf{x},z)]=\frac{1}{p(y)}\underset{i\in Z(y)}{\sum}m(x_i)$

那么只需要确保$\sum_{i\in Z(y)}m(x_i)=1$，即可保证RIS估计无偏。

有很多方法选取$m(x)$，最简单的是直接取$m(x_z)=1/|Z(x_z)|$，也就是取$x_z$处PDF不为零的样本数量的倒数。该方案虽然得到了无偏的结果，但是对PDF倒数的估计可能会产生问题。如果一个候选样本的PDF很接近零，那么据此得到的PDF倒数的RIS估计会有很大的噪声。

第二种选取的方法是结合多重重要性采样，例如：

$m(x_z)=\frac{p_z(x_z)}{\sum_{i=1}^{M}p_i(x_z)}$

上式采用了平衡启发式方法。

### ReSTIR GI

为了能对间接光照应用ReSTIR方法，就必须重新表示间接光照的方向。

我们称相机能直接看到的表面点为**可见点**；对于可见点，可以随机地采样光线方向来寻找最近的相交表面，我们称这些相交点为**采样点**。得到采样点之后就可以构建样本序列、重新采样重要性，最终得到ReSTIR GI结果。

#### 附一：重要性重采样（IS）和重采样重要性采样（RIS）的详细论述

重要性重采样最早由Rubin于1987年提出。

当我们希望生成符合某个PDF $g$ 的样本序列时，可能无法由常规方法（比如CDF求逆）生成，这或许是因为无法求解$g$的闭合形式的解析表示，也可能是因为$g$的积分形式和逆函数很难表达。这种情况下，我们可以通过一种原始分布生成初始样本序列，然后通过合适的权重从样本中抽取单个样本，这就能使得重新抽取的样本序列符合目标PDF $g$ 。

上述方法便是**重要性重采样（IS）**

将重要性重采样和重要性采样结合，即可得到**重采样重要性采样**。

如果只使用重要性采样，则由于理想PDF $g$ 的一些限制（非归一化、难以抽样等）而无法在$g$上进行。这时，可以先从重要性重采样中得到符合理想PDF $g$ 的重要性样本，然后再对该重要性样本序列使用重要性采样，如此便可得到**重采样重要性采样**方法。

用重要性采样书写的RIS形式为：

$\hat{I}_{ris}=\frac{1}{N}\underset{i=1}{\overset{N}{\sum}}w(\mathbf{X}_i,Y_i)\frac{f(Y_i)}{g(Y_i)}$

其中，权重值$w(\mathbf{X}_i,Y_i)$由重要性重采样的均值确定，即：

$w(\mathbf{X}_i,Y_i)=\frac{1}{M}\overset{M}{\underset{j=1}{\sum}}w_{ij}$

表示从样本序列$\mathbf{X}_i$中得到的PDF估计，也就是$w(\mathbf{X}_i,Y_i)\approx \int_{\mathbf{X}_i}g(Y_i)$

由于权重中包含了样本序列$\mathbf{X}_i$的所有PDF值的积分，而重要性采样中又除以了所有PDF值，因此我们可以保证权重值的归一化。

最后整理上式可以得到：

$\hat{I}_{ris}=\frac{1}{N}\underset{i=1}{\overset{N}{\sum}}\left(\frac{f(Y_i)}{g(Y_i)}\cdot\frac{1}{M}\underset{j=1}{\overset{M}{\sum}}w_{ij}\right)$

**方差分析**：

待续，有空再补。

#### 算法流程
每一帧中，对于每个像素：从可见点向随机方向追踪一条光线，记录交点信息（交点的位置、法线、辐射度，出发点的位置和法线）；时域复用样本，将上一步中的信息用于更新时域蓄水池（从蓄水池重投影样本和当前样本中随机选择）；空间复用样本，随机选取邻域像素的时域蓄水池来更新空间蓄水池（比对深度和法线信息，选取几何特征相似的邻域像素）。

初始采样：对于每个像素，从G-buffer中获取位置和法线信息；依pdf随机采样光线方向；追踪光线找到采样点（位置和法线）；蒙特卡洛估计采样点的出射辐射度；将结果写入初始采样的缓冲中。

时域复用：TODO

空间复用：TODO

### ReSTIR PT (GRIS)

早期的ReSTIR方法重用了大量样本，这引入了样本之间的相关性，从而无法保证RIS的收敛性。GRIS推广了ReSTIR理论，使得复用样本也可以正常收敛，该项工作进一步巩固了相关理论的基石，使得ReSTIR方法的方差界和收敛界也可以计算出来。

ReSTIR不稳定的因素在于，作为其根基的RIS理论要求样本之间的贡献都是独立且均等的，重用这些样本直接打破了它们的独立性要求，也使得收敛变缓。ReSTIR、ReSTIR GI的成功应用可以从经验角度解释为，较小的相关性并不会对收敛产生很大影响，但怎样的复用会怎样影响收敛值得彻底地研究，特别是对于更复杂的场景来说，保证样本之间的独立性可能更加重要。

GRIS理论重新阐述了之前从RIS到ReSTIR的理论：

RIS的输入是一组域为 $\Omega$，概率密度函数为 $p$ 的独立同分布的多个随机样本$(X_i)_{i=1}^{M}$。RIS的目标是从输入样本序列中随机抽样生成新的样本序列$Y$，以使得样本序列$Y$的概率密度函数$p_Y$是更适合$\Omega$上被积函数$f$的重要性采样。

更精确地说，我们定义一个非负的目标函数$\hat{p}$，并随机地从中采样，随着采样数$M$的增长，那么得到的$p_Y$会越来越接近归一化的$\hat{p}$。

从算法上看，对于样本$X_i$，我们依概率$w_i/\sum_{j=1}^Mw_j$从多个样本中选取它。其中的$w_i$被称为重抽样权重，定义为$w_i=\hat{p}(X_i)/p(X_i)$。

为了便于表述，我们将重抽样权重除以$M$（因为分号上下都变化了，所以不影响抽取概率），得到新的权重定义：

$w_i=\frac{1}{M}\hat{p}(X_i)W_i\qquad and\quad W_i=\frac{1}{p(X_i)}$

新的权重其实是对$\hat{p}(X_i)$的平均无偏贡献，样本序列$Y$的PDF并不好表述，但可以用它的无偏贡献权重来代替$1/p_Y(Y)$，也就是：

$\frac{1}{p_Y(Y)}=W_Y=\frac{1}{\hat{p}(Y)}\underset{i=1}{\overset{M}{\sum}}w_i$

于是，对于定义域上任意$f>0$都有$p_Y>0$的情况，我们可以得到：

$\int_{\Omega}f(x)dy=\mathbb{E}[f(Y)W_Y]$

当样本序列中的样本来自不同的PDF时，需要使用MIS来进行采样，也就是重采样多重重要性采样，其重要性权值为：

$m_i(x)=\frac{p_i(x)}{\sum_{j=1}^Mp_j(x)}$

### GRIS理论

RIS假定了从相互独立的样本中选取样本。GRIS没有这样的前提，候选样本序列可以是来自于不同域$\Omega_i$的相关样本$(X_i)_{i=1}^M$。GRIS从候选样本序列中随机地选择样本$X_s$，并通过一个移动映射(shift mapping)来将样本映射到被积函数$f$的域上，这样就能使生成的样本序列$Y$的PDF接近于归一化的目标PDF $\bar{p}$ 。

简单来说，GRIS通过一个映射$T$，完成了从任意样本域到被积函数域的映射，也就是修正权重表达式为：

$w_i=m_i(T_i(X_i))\hat{p}(T_i(X_i))W_i\cdot|\partial T_i/\partial X_i|$

新选取的样本应当事先完成映射变换，也就是

$Y=T_s(X_s)$

这样就可以沿用以往的算法流程。

从域$\Omega_i$到域$\Omega$的移动映射$T_i$定义为从子集$\mathcal{D}(T_i)\subset\Omega_i$到图像$\mathcal{I}(T_i)\subset\Omega$的双射函数。

那么贡献函数为：

$g_i(x)=c_i(y)f(y_i)|\frac{\partial T_i}{\partial x}|$

其中，$y_i$是对$T_i(x)$的简写，$c_i$是MIS的贡献权重（可以任取，只需要满足归一化），$|\frac{\partial T_i}{\partial x}|$是$x\rightarrow y_i$的雅可比行列式。最后就能得到：

$\underset{i=1}{\overset{M}{\sum}}\int_{\Omega_i}g_i(x)dx=\int_{\Omega}f(x)dx$

对于$x\notin\mathcal{D}(T_i)$，我们定义其$g_i(x)=0$，同时调整MIS贡献权重$c_i$来补偿。

我们假设权重$w_i$遵循以下规则：

$w_i>0\iff X_i\in\mathcal{D}(T_i)\space\&\space\hat{p}(Y_i)>0$

$w_i=0,\quad otherwise$

也就是，$Y$的支撑集满足：

$\textrm{supp}Y=\textrm{supp}\hat{p}\cap\underset{i=1}{\overset{M}{\bigcup}}T_i(\textrm{supp}X_i)$

上式表明$\textrm{supp}Y\subset\textrm{supp}\hat{p}$，也可以假设$\textrm{supp}\hat{p}\subset\textrm{supp}Y$，这样就可以得到两个支撑集相等，也就是双射关系。
