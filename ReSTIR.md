# ReSTIR

我们将介绍一种适用于动态场景光线追踪的方法来采样大量光源的一次弹射直接光照。该方法基于2005年的重采样重要性采样（RIS）技术，该技术从一种分布中提取一组样本，并使用另一种更接近被积函数的分布来从中选取一个加权子集。本文的不同之处在于，我们使用一个小规模的固定尺寸的“蓄水池”数据结构来存储被接受的样本，并使用关联采样算法（常被用于非图形学的应用中）来达到稳定的实时表现。

蓄水池的数据结构很简单，是一个固定尺寸的数组，但它通过复用时空邻域的统计信息来随机地、渐进地、分层地改进每个像素地直接光照采样的PDF。与实时降噪算法复用时空邻域的像素颜色不同，我们复用了**采样概率**，这也使得我们能得到无偏的结果。我们也给出了一种有偏的方法来进一步降低噪声，代价是在几何不连续的位置附近会更加黯淡。

### 原理

先给出$y$点反射的辐射度表示：

$L(y,\omega)=\int_{A}\rho(y,\overrightarrow{yx}\leftrightarrow\overrightarrow{\omega})L_e(x\to y)G(x\leftrightarrow y)V(x\leftrightarrow y)dA_x$

其中，$A$表示所有的光源表面，$\rho$表示BSDF，$V$表示相互可见性，G是含有距离平方反比和余弦加权的几何项。

简记形式为：$L=\int_Af(x)dx\quad where\space f(x)\equiv\rho(x)L_e(x)G(x)V(x)$

**蒙特卡洛重要性采样**一般表述形式为：

${\left<L\right>}_{is}^N=\frac{1}{N}\underset{i=1}{\overset{N}{\sum}}\frac{f(x_i)}{p(x_i)}\approx L$

其中，$N$为采样数，$x_i$为采样位置，$p(x_i)$为概率密度函数（PDF）。如果在任何$f(x)$不为零的位置，$p(x)$也都为正，则该重要性采样便是无偏的。理想的概率密度函数应当与原函数$f(x)$相关以减少方差。

实际上，直接按原函数比例取样是不现实的，因为会受可见性$V(x)$的影响。但我们仍然可以按被积函数中的部分项（BSDF项和自发光项）的比例取样。给定$M$个候选采样策略$p_s$，则**多重重要性采样**对每种采样策略分别选取$N_s$个样本，并最终得到一个加权估计：

${\left<L\right>}_{mis}^{M,N}=\underset{s=1}{\overset{M}{\sum}}\frac{1}{N_s}\underset{i=1}{\overset{N_s}{\sum}}w_s(x_i)\frac{f(x_i)}{p_s(x_i)}$

其中，权重函数满足归一化$\sum_{s=1}^{M}w_s(x)=1$，因而MIS仍然是无偏估计。

一般使用启发式的权重函数：$w_s(x)=\frac{N_sp_s(x)}{\sum_jN_jp_j(x)}$

另一种可选的MIS采样策略则是按部分项乘积的近似比例采样。**重采样重要性采样**（RIS）就是这么做的，它从次优的原始分布$p$中生成$M\geq1$个候选样本$\mathbf{x}=\left\{x_1,...,x_M\right\}$，然后根据离散概率随机地从中选取一个序号$z\in\left\{1,...,M\right\}$：

$p(z|\mathbf{x})=\frac{\textrm{w}(x_z)}{\sum_{i=1}^M\textrm{w}(x_i)}\quad with\space \textrm{w}(x)=\frac{\hat{p}(x)}{p(x)}$

目标概率密度函数$\hat{p}(x)$可能是不存在实际的采样算法的（比如，$\hat{p}\propto\rho\cdot L_e\cdot G$），则单样本的RIS估计就可以表述为：

${\left<L\right>}_{ris}^{1,M}=\frac{f(y)}{\hat{p}(y)}\cdot\left(\frac{1}{M}\underset{j=1}{\overset{M}{\sum}}\textrm{w}(x_j)\right)$

其中，$y\equiv x_z$

直观上，RIS估计使用圆括号中的项作为系数来修正采样，使得$y$就好像是按$\hat{p}$采样的。当$M=1$是，RIS估计等同于IS。

多次重复RIS并平均结果，就可以得到N样本的RIS估计：

${\left<L\right>}_{ris}^{N,M}=\frac{1}{N}\underset{i=1}{\overset{N}{\sum}}\left(\frac{f(y_i)}{\hat{p}(y_i)}\cdot\left(\frac{1}{M}\underset{j=1}{\overset{M}{\sum}\textrm{w}(x_{ij})}\right)\right)$

只要$p,\hat{p}$在$f$非零处都为正值，则RIS也是无偏估计。简便起见，后续将按$N=1$使用。

一般地，图片中的每个像素$q$都有唯一的被积函数$f_q$以及相应的PDF $\hat{p}_q$

也可以混合使用MIS和RIS，即使用MIS生成初始PDF，再通过RIS优化。但这种方式会使得计算开销随MIS的增加而显著增加（平方倍），我们稍后使用一种全新的MIS方法来解决该问题。

**加权蓄水池采样**

### 时空重用的流式RIS


