# Multiplexed Metropolis Light Transport

除了直接使用蒙特卡洛方法求解渲染方程外，还可以引入一些额外的先验知识来优化计算过程，比较有代表性的就是多重重要性采样（MIS）。另一方面，马尔科夫链蒙特卡洛方法（MCMC）是一种更直接的方法，这类方法在局部范围内应用MCMC来寻找光线路径，新的光路由当前光路的信息生成。虽然原始的MCMC方法不依赖于概率密度（这也是它流行的原因），但本篇论文尝试将MCMC和MIS结合起来，得到了新的方法。

### MCMC

马尔科夫链蒙特卡洛积分方法可以表述为：

$\int_{\Omega}g(\mathbf{x})\approx\frac{b}{N}\underset{i=1}{\overset{N}{\sum}}\frac{g(\mathbf{x}_i)}{p^*(\mathbf{x}_i)}$

当然，纯粹的MCMC并不实用，因为需要计算归一化系数$b=\int_{\Omega}p^*(\mathbf{x})d\mathbf{x}$，但MCMC的优势在于，当需要积分不同函数$\int_{\Omega}g_0(\mathbf{x})d\mathbf{x},\int_{\Omega}g_1(\mathbf{x})d\mathbf{x},...$时，如果它们使用相同的概率密度函数$p^*(\mathbf{x})$，则MCMC就可以使用相同的归一化系数$b$，而只需要额外通过一个原始的蒙特卡洛方法计算一下$b$即可。

MCMC生成一系列历史样本，构成一条马尔科夫链。给定一个当前样本$\mathbf{x}_i$，那么一个提议样本$\mathbf{y}$则是由条件概率密度$q(\mathbf{x}_i\rightarrow\mathbf{y})$产生，并按概率$min(a,1)$接受该样本作为下一个样本，否则仍沿用当前样本作为下一个样本。

$a(\mathbf{x}_i\rightarrow\mathbf{y})=\frac{p^*(\mathbf{y})q(\mathbf{y}\rightarrow\mathbf{x}_i)}{p^*(\mathbf{x}_i)q(\mathbf{x}_i\rightarrow\mathbf{y})}$

上述过程便是Metropolis-Hastings算法。

### 路径积分

光线传输模拟的本质其实是计算路径积分：

$I_j=\int_{\Omega(\mathcal{M})}h_j(\bar{x})f(\bar{x})d\mu(\bar{x})=\underset{k=1}{\overset{\infin}{\sum}}\int_{\Omega^k(\mathcal{M})}h_j(\bar{x})f(\bar{x})d\mu(\bar{x})$

其中，$I_j$是第$j$个像素的强度，$\Omega^k(\mathcal{M})$是所有光路长度为$k$的路径空间，$\Omega(\mathcal{M})=\bigcup_{k=1}^{\infin}\Omega^k(\mathcal{M})$是全部光路的路径空间，$\bar{x}\in\Omega^k(\mathcal{M})$是从光源到相机的一条完整路径。路径可以表示为场景流形$\mathcal{M}$上的一个向量$\bar{x}=(\mathbf{x}_0,\mathbf{x}_1,\mathbf{x}_2,...,\mathbf{x}_k)$。$f(\bar{x})$是路径上各点反射过程的乘积；$h_j(\bar{x})$是一个滤波函数，仅对第$j$个像素的支撑集非零。

Path tracing将蒙特卡洛积分直接应用在了路径积分上，对于像素$I_j$，通过依概率采样路径$\bar{x}_i\sim p_j(\bar{x}_i)$来计算其蒙特卡洛估计：

$I_j\approx\frac{1}{N}\underset{i=1}{\overset{N}{\sum}}\frac{h_j(\bar{x}_i)f(\bar{x}_i)}{p_j(\bar{x}_i)}=\frac{1}{N}\underset{i=1}{\overset{N}{\sum}}h_j(\bar{x}_i)C_j(\bar{x}_i)$

其中，$C_j(\bar{x})=f(\bar{x})/p_j(\bar{x})$是路径贡献。
