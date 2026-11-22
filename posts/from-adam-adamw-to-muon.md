这篇笔记整理自 Codex 的两段会话：LLM 项目里的「解释训练优化器」和 reading 项目里的「解释 Muon 优化器」。前者从显存和 AdamW 的解耦讲起，后者沿着 Muon 的矩阵正交化展开。把两条线放在一起看，优化器的发展其实一直在回答同一个问题：**拿到梯度之后，应该如何构造参数更新量？**

SGD 的答案是直接用梯度；Adam 的答案是按每个参数的历史梯度做自适应缩放；AdamW 的答案是把任务更新和权重衰减分开；Muon 则把视角从单个参数抬高到整个二维权重矩阵，开始调整不同奇异方向的更新强度。

## 一、先统一视角：优化器在构造什么

几乎所有一阶优化器都可以写成

```math
\theta_{t+1}=\theta_t-\eta\,u_t
```

其中 $\theta_t$ 是参数，$\eta$ 是学习率，$u_t$ 是优化器根据梯度历史构造出来的更新方向。不同优化器的差别，主要就在 $u_t$ 怎么来：

- SGD：$u_t=g_t$，直接使用当前梯度。
- Momentum：先对梯度做指数移动平均，减少单批梯度的抖动。
- Adam / AdamW：同时维护一阶矩和二阶矩，对每个参数做逐元素自适应缩放。
- Muon：对二维矩阵的动量做近似正交化，再按矩阵几何结构更新。

所以优化器并不是「让损失自动下降」的黑盒。它决定的是：**同样一份梯度，如何被重新加权、平滑、缩放和约束。**

## 二、从 SGD 到 Adam：为什么需要逐参数缩放

最简单的 SGD 更新是

```math
\theta_{t+1}=\theta_t-\eta g_t
```

它对所有参数使用同一个学习率。问题是，不同参数的梯度尺度可能相差很大：Embedding 的稀疏梯度、LayerNorm 的缩放参数、Attention 的投影矩阵，梯度方差往往并不相同。一个统一学习率很难同时适合所有参数。

Momentum 先解决「方向抖动」：

```math
m_t=\beta m_{t-1}+(1-\beta)g_t
```

这里采用 $m_t$ 是梯度的指数移动平均这一常见约定；有些实现会写成 $m_t=\beta m_{t-1}+g_t$，只是整体缩放约定不同。动量让更新方向在一段时间内保持连续，减少单次 mini-batch 带来的随机波动。

Adam 进一步引入二阶矩：

```math
\begin{aligned}
m_t&=\beta_1m_{t-1}+(1-\beta_1)g_t\\
v_t&=\beta_2v_{t-1}+(1-\beta_2)g_t^2\\
\hat m_t&=\frac{m_t}{1-\beta_1^t},\qquad
\hat v_t=\frac{v_t}{1-\beta_2^t}\\
\theta_{t+1}&=\theta_t-\eta\frac{\hat m_t}{\sqrt{\hat v_t}+\epsilon}
\end{aligned}
```

$m_t$ 记录梯度的方向，$v_t$ 记录梯度平方的历史尺度。分母 $\sqrt{\hat v_t}+\epsilon$ 会根据每个参数自己的历史梯度动态调整步长：梯度一直很大的参数，更新会被压小；梯度一直很小的参数，更新会被相对放大。

偏差修正 $1-\beta_1^t$ 和 $1-\beta_2^t$ 也很重要。训练刚开始时，$m_0$、$v_0$ 都是 0，如果不做修正，前几步的矩估计会系统性偏小。偏差修正让 Adam 在早期也能使用接近真实尺度的统计量。

这也解释了 Adam 的一个代价：它需要为每个参数保存两个状态。

## 三、优化器状态到底占多少显存

假设模型有 $N$ 个参数，并且优化器状态使用 FP32：

- SGD：不保存额外历史状态，约 $0$ 字节/参数。
- Momentum：保存一个动量 $m$，约 $4$ 字节/参数。
- Adam / AdamW：保存 $m$ 和 $v$ 两个状态，约 $8$ 字节/参数。

以 7B 参数模型为例，AdamW 单独的状态就约 $7\times10^9\times8=56$ GB。这里还没有计算参数本身、梯度、激活值、临时 buffer 和分布式训练中的副本。混合精度训练通常也不会把优化器状态自动变成 FP16：为了保证数值稳定，$m$ 和 $v$ 往往仍然使用 FP32。

所以「AdamW 很吃显存」不是实现细节，而是它的算法定义带来的结构性成本。Muon 只维护一个动量状态，在状态显存上确实更接近 Momentum；但它又增加了 Newton-Schulz 迭代和矩阵乘法，不能简单说成「全面更省」。

## 四、AdamW：为什么要把权重衰减拆出来

要理解 AdamW，先要理解权重衰减和 L2 正则化的区别。

在普通 SGD 中，如果我们在损失中加入 L2 正则：

```math
L_{\text{total}}(\theta)=L_{\text{task}}(\theta)+\frac{\lambda}{2}\|\theta\|_2^2
```

梯度会多出一项 $\lambda\theta$：

```math
g'_t=g_t+\lambda\theta_t
```

把它代入 SGD：

```math
\begin{aligned}
\theta_{t+1}
&=\theta_t-\eta(g_t+\lambda\theta_t)\\
&=(1-\eta\lambda)\theta_t-\eta g_t
\end{aligned}
```

这正是「先让参数乘上 $1-\eta\lambda$，再沿任务梯度更新」。所以在 SGD 中，L2 正则化和权重衰减在代数上等价。

但 Adam 不是简单地乘学习率。若把 $\lambda\theta_t$ 加进梯度再传给 Adam，它会同时进入一阶矩和二阶矩：

```math
\begin{aligned}
m_t&=\beta_1m_{t-1}+(1-\beta_1)(g_t+\lambda\theta_t)\\
v_t&=\beta_2v_{t-1}+(1-\beta_2)(g_t+\lambda\theta_t)^2
\end{aligned}
```

平方项展开后会出现 $2\lambda\theta_tg_t$ 和 $\lambda^2\theta_t^2$。也就是说，正则化不仅改变更新方向，还会污染二阶矩，并和任务梯度产生交叉项。最终，衰减项还会被 Adam 的自适应分母缩放。

把 Adam 的自适应系数记成

```math
D_t=\frac{1}{\sqrt{\hat v_t}+\epsilon}
```

那么 Adam + L2 的衰减部分近似是

```math
-\eta D_t\lambda\theta_t
```

而真正解耦的权重衰减希望得到的是

```math
-\eta\lambda\theta_t
```

两者一般不相等，因为 $D_t$ 会随参数和历史梯度变化。**AdamW 的关键不是换了一个名字，而是把「任务梯度更新」和「参数衰减」从同一个自适应机制里拆开。**

AdamW 的更新可以写成

```math
\theta_{t+1}
=\theta_t
-\eta\frac{\hat m_t}{\sqrt{\hat v_t}+\epsilon}
-\eta\lambda\theta_t
```

其中 $m_t$、$v_t$ 只由任务梯度 $g_t$ 更新，不再把 $\lambda\theta_t$ 放进矩估计。

一个单参数例子可以看得很清楚。设 $\theta=1$、$\eta=0.01$、$\lambda=0.1$，当前任务梯度为 0：

- Adam + L2：正则化梯度是 $0.1$，经过 Adam 的归一化后更新量约为 $0.01$，参数变成 $0.99$。
- AdamW：任务梯度更新为 0，只剩独立衰减 $-0.01\times0.1\times1=-0.001$，参数变成 $0.999$。

同样的学习率和衰减系数，实际衰减幅度相差很大。这就是「解耦」的工程意义。这里是一个首次更新、忽略 $\epsilon$ 的教学例子，但它揭示了核心机制：Adam 的自适应分母会改变 L2 惩罚的实际强度。
## 五、Muon：从逐参数缩放走向矩阵几何

AdamW 仍然主要在逐元素层面工作：每个参数有自己的 $m$ 和 $v$，更新量大体上是逐坐标构造的。对于二维权重矩阵，这种处理会忽略一个重要结构：矩阵的不同奇异方向，可能承载着强弱差异很大的更新。

Muon 的全称是 **MomentUm Orthogonalized by Newton-Schulz**。它的简化流程是：

```math
\begin{aligned}
G_t&=\nabla_W L(W_t)\\
M_t&=\beta M_{t-1}+G_t\\
U_t&\approx\operatorname{Ortho}(M_t)\\
W_{t+1}&=W_t-\eta U_t
\end{aligned}
```

这里 $W_t$ 是二维权重矩阵，$M_t$ 是动量矩阵，$\operatorname{Ortho}(\cdot)$ 表示把矩阵近似正交化。实际实现还会包含 Nesterov 风格动量、形状缩放和数值归一化；上面的式子先保留机制主线。

对矩阵做奇异值分解：

```math
M=U\Sigma V^\top
```

其中 $\Sigma$ 的对角元素是奇异值。理想的正交化会保留左右奇异向量，把非零奇异值都替换为 1：

```math
\operatorname{Ortho}(M)=UV^\top
```

从几何上看，$UV^\top$ 是一个距离 $M$ 最近的半正交矩阵：它保留矩阵的主要方向结构，却不再让少数大奇异值支配整个更新。

Muon 的动机可以理解成：Transformer 隐藏层的更新矩阵往往是高条件数的，少数方向贡献了大部分更新幅度。正交化会压低这些方向的相对优势，让其他非零奇异方向也能参与学习。这里要注意两点：

- 这是对更新矩阵的处理，不是对权重矩阵本身做正交化。
- 正交化不会凭空创造梯度中不存在的方向；它改变的是已有方向之间的相对强度。

## 六、为什么用 Newton-Schulz，而不是直接 SVD

如果每一步都对大型矩阵做完整 SVD，代价会很高。Muon 使用 Newton-Schulz 迭代近似正交化。一个常见的迭代形式是

```math
\begin{aligned}
X_0&=\frac{M}{\|M\|_F+\epsilon}\\
A_k&=X_kX_k^\top\\
X_{k+1}&=aX_k+bA_kX_k+cA_k^2X_k
\end{aligned}
```

论文博客给出的系数是

```math
a=3.4445,\qquad b=-4.7750,\qquad c=2.0315
```

为什么这个迭代会走向正交化？如果当前矩阵可以写成 $X_k=U\Sigma_kV^\top$，那么迭代只会改变奇异值，不改变左右奇异向量。定义多项式

```math
\varphi(s)=as+bs^3+cs^5
```

每一步实际上是在做 $s_{k+1}=\varphi(s_k)$。合适的系数会让归一化后的奇异值逐渐靠近 1，于是 $X_k$ 近似变成 $UV^\top$。

实际实现通常会在矩阵较「高」时先转置，以减少矩阵乘法开销；有限次迭代得到的是近似半正交矩阵，不是精确的 SVD 结果。因此 Muon 的收益和开销都需要放在具体形状、批量大小和硬件上衡量。

## 七、AdamW 和 Muon 到底怎么选

两者最核心的区别可以写成一句话：

> AdamW 关心「每个参数应该更新多少」；Muon 关心「整个二维更新矩阵的不同奇异方向应该如何分配强度」。

工程上通常不是二选一，而是混合使用：

- 隐藏层中的二维权重矩阵：使用 Muon。
- Embedding、最终输出头、偏置、LayerNorm/RMSNorm 等参数：继续使用 AdamW 或其他标准优化器。
- Transformer 的 Q、K、V 投影：原博客的经验是分别优化通常更好，但是否合并要由实验决定。

这不是「Muon 全面替代 AdamW」的协议，而是参数分组策略。原因很直接：Muon 的正交化针对矩阵结构设计，Embedding、输出层和一堆一维参数的几何意义并不相同。

实践中要注意：

- 学习率、动量、权重衰减和参数分组都要重新调，不能直接沿用 AdamW 的超参数。
- Muon 每一步多了 Newton-Schulz 迭代和矩阵乘法，单步可能更慢。
- 它只维护一个动量状态，优化器状态显存通常少于 AdamW 的两个状态，但临时矩阵和算子开销仍然存在。
- 比较时既要看达到目标 loss 需要多少步，也要看真实 wall-clock 时间。

## 八、实验结论应该怎么读

Muon 原博客报告过一些有代表性的结果：CIFAR-10 达到 94% 准确率所需的 A100 秒数从 3.3 降到 2.6；NanoGPT 训练速度约提升 1.35 倍；在 1.5B Transformer 实验中，达到目标 HellaSwag 性能约需 8 小时 8×H100，而调优后的 AdamW 约需 13.3 小时；Newton-Schulz 带来的额外 FLOPs 在作者典型场景中通常低于 1%。

这些结果说明 Muon 值得认真研究，但不能直接推广成「所有模型、所有任务都会更快」。至少有三件事需要分开看：

1. 样本效率：达到同样效果需要多少 token 或更新步。
2. 计算效率：每一步多花了多少矩阵运算。
3. 最终 wall-clock：训练系统、并行策略、kernel 实现都会影响结果。

作者也明确提醒，和 Muon 比较的 AdamW 必须是充分调优的基线。拿一个没有调好的 AdamW 做对比，不足以证明新优化器更好。

## 九、几个容易混淆的点

- 「AdamW 就是 Adam 加 L2。」不对。AdamW 的关键是让权重衰减不进入 Adam 的一阶、二阶矩。
- 「Muon 是对权重做正交化。」不对。它正交化的是梯度动量形成的更新矩阵。
- 「正交化会让每个参数更新一样大。」不对。它调整的是矩阵奇异方向的相对强度，不是把所有元素变成同一个数。
- 「Muon 只有一个状态，所以一定更省显存、更快。」不一定。它省的是优化器状态，但会增加 Newton-Schulz 的计算和临时内存。
- 「Muon 已经在某个基准上更快，所以可以直接替换 AdamW。」不成立。参数分组、学习率、模型结构和训练系统都要重新验证。

## 十、把这条线串起来

- **SGD**：直接用当前梯度。
- **Momentum**：对梯度做指数移动平均，平滑更新方向。
- **Adam**：一阶矩加二阶矩，做逐参数自适应缩放。
- **AdamW**：把权重衰减从自适应更新中解耦。
- **Muon**：对二维动量矩阵做近似正交化，调整不同奇异方向的更新强度。

如果只记一句话：**AdamW 解决的是「每个坐标的尺度如何自适应」，Muon 进一步问的是「整个矩阵的更新几何是否合理」。**

## 参考资料

- Kingma & Ba, *Adam: A Method for Stochastic Optimization*, 2014. [arXiv:1412.6980](https://arxiv.org/abs/1412.6980)
- Loshchilov & Hutter, *Decoupled Weight Decay Regularization*, 2017. [arXiv:1711.05101](https://arxiv.org/abs/1711.05101)
- Keller Jordan, *Muon: An optimizer for hidden layers in neural networks*, 2024. [Blog](https://kellerjordan.github.io/posts/muon/)
- Liu et al., *Muon is Scalable for LLM Training*, 2025. [arXiv:2502.16982](https://arxiv.org/abs/2502.16982)

---

这篇笔记的来源是 Codex 的「解释训练优化器」与「解释 Muon 优化器」两段会话，并补入了 Adam、AdamW 和 Muon 原始资料的公式与边界说明。公式是机制级简化，具体实现仍应以对应代码和实验配置为准。