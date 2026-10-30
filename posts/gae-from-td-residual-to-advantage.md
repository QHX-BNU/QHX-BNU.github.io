这篇笔记整理自我在 Codex 的 LLM 项目里围绕 GAE、TD(λ)、baseline、PPO 和 GRPO 的一串讨论。最开始我总把 GAE 记成一个「带 λ 的公式」：知道它能调偏差和方差，却说不清它到底在估什么，也说不清为什么 `λ` 小不等于只看短期奖励。

后来我把这条线重新走了一遍：从强化学习目标出发，经过策略梯度、baseline、TD 残差和多步回报，最后再看 PPO 与 GRPO。真正有用的结论是：

> **GAE 有两条等价主线：它既是不同步长优势估计的几何混合，也是对「价值塑形后的奖励」做 γλ 折扣的蒙特卡洛回报。**

把这两条线同时记住，很多公式就不再是硬背的了。

## 一、先把问题摆正：我们到底在估什么

强化学习优化的不是某一次轨迹的得分，而是策略的期望累积回报。设策略为 `π_θ`，轨迹为 `τ`，目标是

```math
J_\gamma(\theta)=\mathbb{E}_{\tau\sim p_\theta}\left[\sum_{t=0}^{T-1}\gamma^t r_t\right]
```

其中 `γ` 既决定不同延迟奖励的权重，也定义了后面价值函数所对应的折扣目标。为了评价「在某个状态下采取某个动作到底好不好」，定义

```math
\begin{aligned}
G_t^\gamma &= \sum_{l=0}^{T-t-1}\gamma^l r_{t+l}\\
V^\pi(s) &= \mathbb{E}_\pi[G_t^\gamma\mid s_t=s]\\
Q^\pi(s,a) &= \mathbb{E}_\pi[G_t^\gamma\mid s_t=s,a_t=a]\\
A^\pi(s,a) &= Q^\pi(s,a)-V^\pi(s)
\end{aligned}
```

可以这样理解：

- `V(s)`：从状态 `s` 开始，按当前策略行动，平均能拿多少回报。
- `Q(s,a)`：先执行动作 `a`，之后再按当前策略行动，平均能拿多少回报。
- `A(s,a)`：这个动作比当前策略的平均选择好多少。

优势不是「即时奖励」，也不是「相对最优动作差多少」。它比较的是**同一个状态下，这个动作与当前策略的平均表现**。因此，在当前策略下对动作取期望，优势的期望为零：

```math
\mathbb{E}_{a\sim\pi(\cdot\mid s)}[A^\pi(s,a)]=0
```

策略梯度的核心，就是用「动作概率的对数梯度」乘以一个评价信号：

```math
\begin{aligned}
\nabla J_\gamma(\theta)
&=\mathbb{E}\left[\sum_t\gamma^t\nabla\log\pi_\theta(a_t\mid s_t)\,(G_t^\gamma-b(s_t))\right]
\end{aligned}
```

这里的 `b(s_t)` 是 baseline。为什么可以减掉它？这就是理解 GAE 之前必须先过的一关。

## 二、baseline：减掉的是梯度噪声，不是奖励本身

对固定状态 `s`，策略的 score 有一个重要性质：

```math
\mathbb{E}_{a\sim\pi_\theta(\cdot\mid s)}[\nabla\log\pi_\theta(a\mid s)]=0
```

因为

```math
\begin{aligned}
\sum_a\pi_\theta(a\mid s)\nabla\log\pi_\theta(a\mid s)
&=\nabla\sum_a\pi_\theta(a\mid s)\\
&=\nabla 1=0
\end{aligned}
```

所以，只要 `b(s)` 在给定状态后不依赖当前采样动作，就有

```math
\mathbb{E}[\nabla\log\pi_\theta(a\mid s)\,b(s)\mid s]=0
```

减去 baseline 不会改变梯度的期望，却可能改变它的方差。这和「对一组奖励减同一个常数，奖励方差不变」并不矛盾：策略梯度用的是 `score × 奖励` 这个乘积，而不是单独的奖励。`score` 会随动作变化，奖励也往往与动作相关，两者存在统计关联，减去 `b(s) · score` 就可能抵消一部分波动。

一个能直接算完的小例子：设动作 `a∈{0,1}`，`p(a=1)=1/2`，奖励 `R=a`，以 logit 为参数时 `score` 为 `a-1/2`。

```text
动作 0：概率 1/2，score = -1/2，R = 0
动作 1：概率 1/2，score =  1/2，R = 1

无 baseline：
  样本梯度分别为 0 和 1/2
  均值 = 1/4，方差 = 1/16

取 b = 1/2：
  两个样本梯度都变成 1/4
  均值仍是 1/4，方差变成 0
```

奖励 `R` 和 `R-b` 的方差相同，但 `score·R` 与 `score·(R-b)` 的方差可以不同。baseline 起的是控制变量的作用。

不过，baseline 也不是「随便选一个就一定更好」。错误或尺度不合适的 baseline 可能放大噪声。价值函数 `V(s)` 是很自然的选择，但在某些方差度量下，最优 baseline 未必严格等于 `V(s)`。实践中最重要的是：baseline 应当与当前动作条件独立，并且不要通过 actor 的梯度反传去更新它。

## 三、从完整回报到 TD 残差

蒙特卡洛方法直接用完整回报 `G_t` 估计优势：

```math
\hat A_t=G_t^\gamma-V(s_t)
```

它保留了完整实际奖励，但一条轨迹后面的随机因素太多，梯度噪声可能很大。另一个极端是只用一步实际转移，然后用价值函数预测未来：

```math
\delta_t=r_t+\gamma V(s_{t+1})-V(s_t)
```

`δ_t` 叫 TD 残差，可以读成：

> 执行动作、观察到下一步之后，对长期回报的评价，比行动前的预期提高了多少。

如果 `V` 正好等于真实价值 `V^π`，由 Bellman 等式，

```math
\begin{aligned}
\mathbb{E}[\delta_t\mid s_t,a_t]
&=Q^\pi(s_t,a_t)-V^\pi(s_t)\\
&=A^\pi(s_t,a_t)
\end{aligned}
```

也就是说，**价值函数精确时，一步 TD 残差已经可以无偏估计当前动作的优势**。它不是「必须等到最后才知道结果」；价值函数已经把这个动作的未来影响压缩进了 `V(s_{t+1})`。

现实中的 `V` 通常不精确。此时只看一步会过度依赖当前预测；多看几步实际奖励，可以修正价值误差。于是有了 k 步优势估计：

```math
\begin{aligned}
\hat A_t^{(k)}
&=\sum_{l=0}^{k-1}\gamma^l\delta_{t+l}^V\\
&=\sum_{l=0}^{k-1}\gamma^l r_{t+l}+\gamma^k V(s_{t+k})-V(s_t)
\end{aligned}
```

第二行来自望远镜消去：中间那些 `γV` 项会相互抵消。`k=1` 时它退化成 TD 残差；`k` 接近完整回合时，它接近「蒙特卡洛回报减 baseline」。不同的 `k` 就是在「更相信实际采样」与「更早相信价值预测」之间移动。

## 四、GAE 的第一条主线：对不同步长做几何混合

GAE 不固定选择一个 `k`，而是把所有 k 步估计按几何权重混合：

```math
\hat A_t^{\mathrm{GAE}(\gamma,\lambda)}
=(1-\lambda)\sum_{k=1}^{\infty}\lambda^{k-1}\hat A_t^{(k)}
```

其中 `0 ≤ λ < 1`。把 k 步估计展开后，某个残差 `δ_{t+l}` 会出现在所有 `k ≥ l+1` 的估计里，它的总系数是

```math
\begin{aligned}
&(1-\lambda)\gamma^l\sum_{k=l+1}^{\infty}\lambda^{k-1}\\
&=(\gamma\lambda)^l
\end{aligned}
```

因此得到最常见的 GAE 形式：

```math
\hat A_t^{\mathrm{GAE}(\gamma,\lambda)}
=\sum_{l=0}^{\infty}(\gamma\lambda)^l\delta_{t+l}^V
```

实际实现只需要从后往前递推：

```math
\hat A_t=\delta_t+\gamma\lambda\hat A_{t+1}
```

真正终止时令 `Â_T = 0`。如果有时间上限截断而不是环境终止，需要单独处理 bootstrap；不能把「截断」与「终止」混为一谈。

### 几何权重到底在做什么？

把归一化权重写成

```math
w_k=(1-\lambda)\lambda^{k-1}
```

它可以看成一个几何分布：以概率 `1-λ` 停止增加实际步数，以概率 `λ` 再多看一步。若随机步长为 `K`，则

```math
\begin{aligned}
\mathbb{E}[K]&=\frac{1}{1-\lambda}\\
\Pr(K\ge k)&=\lambda^{k-1}
\end{aligned}
```

例如 `λ=0.95` 时，一步估计的单项权重只有 `0.05`，但平均会混合 `20` 步，而且 `k≥20` 的尾部总权重约为 `0.377`。所以：

> **「一步的单项权重大」不等于「只依赖一步预测」；长步长的权重需要合起来看。**

### 有限回合的边界

如果距离真正终止还有 `N=T-t` 步，有限回合的混合不能简单截断成 `(1-λ)Σ_{k=1}^{N} λ^{k-1}`，否则权重之和不足 1。最后一项要接住剩余的全部权重：

```math
\begin{aligned}
\hat A_t^{\mathrm{GAE}}
&=(1-\lambda)\sum_{k=1}^{N-1}\lambda^{k-1}\hat A_t^{(k)}
+\lambda^{N-1}\hat A_t^{(N)}
\end{aligned}
```

这是实现和推导时很容易漏掉的一个细节。`λ=1` 可以直接由残差和定义，也可以理解为有限混合在边界上的极限；但不能把无限和中的 `(1-λ)` 单独代成零。

## 五、GAE 的第二条主线：塑形奖励的蒙特卡洛回报

这是我觉得最有解释力的一步。定义塑形奖励

```math
\tilde r_t=r_t+\gamma V(s_{t+1})-V(s_t)
```

它恰好就是 TD 残差：

```math
\tilde r_t=\delta_t^V
```

于是 GAE 可以改写成

```math
\hat A_t^{\mathrm{GAE}(\gamma,\lambda)}
=\sum_{l=0}^{T-t-1}(\gamma\lambda)^l\tilde r_{t+l}
```

右边是什么？就是**沿一条实际轨迹，对塑形奖励 `r̃` 做 `γλ` 折扣后得到的蒙特卡洛回报**。

所以，GAE 并不只有在 `λ=1` 时才是蒙特卡洛估计。对任意 `λ`，它都可以被解释为「先对奖励做价值塑形，再用 `γλ` 做折扣的轨迹回报」。`λ=1` 的特殊之处，是这时还有进一步的化简：

```math
\begin{aligned}
\sum_{l=0}^{N-1}\gamma^l\tilde r_{t+l}
&=G_t^\gamma-V(s_t)+\gamma^N V(s_T)
\end{aligned}
```

在真正终止且 `V(s_T)=0` 时，

```math
\begin{aligned}
\hat A_t^{\mathrm{GAE}(\gamma,1)}
&=G_t^\gamma-V(s_t)
\end{aligned}
```

也就是「原始奖励的完整折扣回报减去 baseline」。

### 为什么塑形可以让信用分配更短？

如果 `V=V^π` 且动作按当前策略采样，那么当前状态的残差期望是优势：

```math
\mathbb{E}[\delta_t\mid s_t,a_t]=A^\pi(s_t,a_t)
```

而对所有后续时刻 `l≥1`，后续动作重新按策略采样，由迭代期望可得

```math
\mathbb{E}[\delta_{t+l}\mid s_t,a_t]=0
```

因此，在有限回合中（或无穷和满足可积条件时），只要价值函数精确，无论 `λ` 取多少，

```math
\mathbb{E}\left[\sum_l(\gamma\lambda)^l\delta_{t+l}\mid s_t,a_t\right]
=A^\pi(s_t,a_t)
```

直观上：**精确的价值函数已经把当前动作的长期影响压缩进了当前残差；后面的残差在条件均值上不再补充当前动作的优势。** 这就是「塑形后可以更积极地折扣远处信号」的原因。它不是说环境奖励被提前发放了，也不是说每条轨迹上的后续残差都真的为零，而是在条件期望层面成立。

当价值函数不精确时，保留一些后续残差可以修正预测误差。`λ` 就在这两种情况之间调节。

这里有一个必须注意的边界：潜势型奖励塑形 `r + γΦ(s') - Φ(s)` 在折扣与边界条件匹配时，可以保留原任务的最优策略结构；但 GAE 使用的是 `γλ` 折扣。`λ<1` 时，不能不加说明地把「塑形不改变原任务」直接套到所有目标上。**塑形蒙特卡洛解释对任意 λ 成立，不代表对原任务优势无条件无偏。**

## 六、γ 和 λ 不是一回事

很多混淆来自把 `γ` 和 `λ` 都当成「折扣」。

- `γ` 定义原始奖励的折扣方式，也定义价值函数要预测的目标。
- `λ` 控制不同步长估计的混合比例，或者等价地控制对塑形奖励施加的额外折扣。

即使两个设置的乘积 `γλ` 相同，GAE 通常也不同，因为 TD 残差本身依赖 `γ`：

```math
\delta_t=r_t+\gamma V(s_{t+1})-V(s_t)
```

例如 `(γ,λ)=(0.99,0.95)` 与 `(0.95,0.99)` 的乘积一样，但它们对应的原始折扣回报和价值目标并不一样，不能互换。

另一个常见误解是「几何衰减是训练收敛的保证」。应该区分三件事：

1. 回报和式是否有定义。有限回合可以允许 `γ=λ=1`；无限时域通常需要 `γλ<1` 之类的条件。
2. 策略评价算子是否收缩。`γ<1` 为折扣 MDP 的 Bellman 算子提供收缩结构。
3. 学习算法是否收敛。还取决于步长、采样、函数逼近、目标非平稳性和优化过程。

所以，`γλ<1` 会让无穷和与递推更稳定、更好分析，但它不是神经网络策略训练必然收敛的证明。

## 七、一个数值例子：两种视角算同一个 GAE

考虑三步轨迹，令 `γ=1`，奖励为

```math
(r_0,r_1,r_2)=(0,0,1)
```

近似价值为

```math
(V(s_0),V(s_1),V(s_2),V(s_3))=(0.4,0.6,0.8,0)
```

先算 TD 残差：

```math
\begin{aligned}
\delta_0 &= 0+0.6-0.4=0.2\\
\delta_1 &= 0+0.8-0.6=0.2\\
\delta_2 &= 1-0.8=0.2
\end{aligned}
```

虽然真实奖励只在最后出现，价值预测的逐步修正让每一步都产生了 0.2 的残差。于是

```math
\hat A_0^{\mathrm{GAE}}=0.2+0.2\lambda+0.2\lambda^2
```

取不同的 `λ`：

- `λ = 0.0`：`Â_0 = 0.200`，只取当前 TD 残差
- `λ = 0.5`：`Â_0 = 0.350`
- `λ = 0.9`：`Â_0 = 0.542`
- `λ = 1.0`：`Â_0 = 0.600`，完整回报 1.0 减去 baseline 0.4

`λ=0.9` 时，有限步长混合的权重是 `(0.1, 0.09, 0.81)`，所以

```math
0.1\times0.2+0.09\times0.4+0.81\times0.6=0.542
```

这和「对塑形奖励 `(0.2,0.2,0.2)` 做 0.9 折扣的回报」完全一致。这个例子说明了 GAE 如何把稀疏终局奖励变成更局部的评价修正，但它不能证明这些价值预测正确，也不能证明 `λ=0.9` 就是最优参数。

## 八、偏差—方差：必须先说明「相对于什么」

如果价值函数精确，那么一步优势估计已经无偏；此时增大 `λ` 主要是在加入更多均值为零的后续噪声。反过来，如果价值函数不精确，多步残差就可能修正 bootstrap 误差。这就是 GAE 的偏差—方差权衡。

但不能把这件事说成「λ 越小，方差一定越小」。条件方差的完整形式包含不同时刻残差之间的协方差：

```math
\begin{aligned}
\operatorname{Var}(\hat A_t\mid s_t,a_t)
&=\sum_{l,m}(\gamma\lambda)^{l+m}
\operatorname{Cov}(\delta_{t+l},\delta_{t+m}\mid s_t,a_t)
\end{aligned}
```

只有在额外条件下，例如 on-policy、价值精确、残差具有鞅差性质时，才能得到更简洁的结论：

```math
\begin{aligned}
\operatorname{Var}(\hat A_t\mid s_t,a_t)
&=\operatorname{Var}(\delta_t\mid s_t,a_t)\\
&\quad+\sum_{l\ge1}(\gamma\lambda)^{2l}
\mathbb{E}[(\delta_{t+l})^2\mid s_t,a_t]
\end{aligned}
```

即便有了这个结论，它也仍然只是条件优势方差，不能直接等同于完整轨迹梯度方差或最终训练表现。

实践上更值得看的是：critic 的解释方差、优势分布是否漂移、梯度噪声、策略 KL、回报曲线，以及不同随机种子下的稳定性。只画一条更平滑的 baseline 曲线，不能证明 GAE 更好；如果实验里根本没有 GAE 曲线，就更不能从 baseline 的稳定性直接推出 GAE 的结论。

## 九、从 GAE 到 PPO：估计信号和控制更新是两件事

GAE 回答的是：

> 这个动作比当前策略的平均表现好多少？

PPO 回答的是另一个问题：

> 拿到这个优势信号后，怎样利用旧策略数据多更新几次，同时避免新策略跑得太远？

PPO-Clip 的常用目标可以写成

```math
\begin{aligned}
\rho_t(\theta)&=\frac{\pi_\theta(a_t\mid s_t)}{\pi_{\mathrm{old}}(a_t\mid s_t)}\\
L^{\mathrm{CLIP}}(\theta)
&=\mathbb{E}_{\mathrm{old}}\left[\min\left(
\rho_t(\theta)\hat A_t,\,
\operatorname{clip}(\rho_t(\theta),1-\epsilon,1+\epsilon)\hat A_t
\right)\right]
\end{aligned}
```

当优势为正时，提高动作概率通常有利，但概率比超过上界后，这个样本不再因为继续提高概率而获得额外收益；当优势为负时，降低动作概率通常有利，但低于下界后也不再获得额外收益。

这里有三个容易说过头的地方：

- PPO-Clip 不是严格的 KL 约束，它不能保证每次更新后的概率比都在区间内。
- 裁剪目标不大于未裁剪代理目标，不等于真实任务回报有相同的下界。
- 裁剪不能修正错误的优势方向；优势估计和更新约束需要共同工作。

GAE 和 PPO 也不是绑定的。PPO 的目标本身不要求优势一定来自 GAE；GAE 原论文的主要实验结合的是 TRPO。只是后来「PPO + GAE」成了非常常见的 actor–critic 组合。

在 LLM 训练里还要区分两个策略：

- `π_old`：当前这批回答的采样策略，用来构造概率比，控制局部更新。
- `π_ref`：通常固定更久的参考模型，用 KL 正则约束整体漂移。

局部更新很小，不代表很多轮之后仍然接近参考模型。这两个参照不能混用。

## 十、从 PPO 到 GRPO：用同题多次采样替代 critic

原始 GRPO 的出发点很直接：对同一个问题采样多份回答，用组内相对奖励构造 baseline，从而省掉一个学习型价值模型。

对问题 `q` 采样 `G` 份回答 `o_1,...,o_G`，得到奖励 `R_1,...,R_G`，组均值为 `R̄`，组标准差为 `s_R`。结果监督版本的优势是

```math
\hat A_{i,t}=\frac{R_i-\bar R}{s_R}
```

并且这个标量会赋给第 `i` 份回答的所有 token。它仍然使用 PPO 风格的裁剪目标，并通常加入相对参考策略的 KL 正则。

所以，「GRPO 用到了 GAE 吗？」的标准回答是：

> **DeepSeekMath 的标准 GRPO 优势构造不依赖 TD 残差与 GAE，而是使用同题组内相对奖励。**

这不排斥后续混合方法引入价值模型或其他优势估计器。

### 组内 baseline 为什么有意义？

如果奖励可以写成「题目难度偏移 + 回答质量差异」：

```math
R(q,o)=c(q)+u(q,o)
```

那么组内中心化会把与题目相关的 `c(q)` 消掉，只保留同一问题下不同回答之间的差异。这样可以避免拿简单题的高分回答和困难题的低分回答直接比较。

但它和状态价值 baseline 的条件信息不同：

- `V(q,o_{<t})` 可以利用生成到当前位置的具体前缀。
- `R̄(q)` 主要利用同一问题下多份完整回答的统计。

前者需要学习跨样本泛化，后者需要花费同题重复采样预算。两者不是简单的「有 critic / 没 critic」的区别。

### 三个容易忽略的细节

第一，组均值包含当前样本自己。若只做中心化、暂时忽略标准差、长度权重、裁剪和 KL，固定 `G` 时会得到一个 `(G-1)/G` 的常数缩放。它不一定破坏学习，但不能直接套用「baseline 与当前动作独立」的无偏证明。使用 leave-one-out baseline 会更干净：

```math
b_{-i}=\frac{1}{G-1}\sum_{j\ne i}R_j
```

第二，除以组标准差不只是「降方差」。标准差是随机量，又依赖当前回答；在大组近似下，它相当于按问题的奖励离散程度重新加权，会改变不同问题、不同结果样本的更新幅度。它不是对原始期望梯度的简单无偏变换。

第三，全对或全错的组没有相对优势。若单次成功概率为 `p`，组内同时出现成功与失败的概率是

```math
P_{\mathrm{mixed}}=1-p^G-(1-p)^G
```

例如 `G=8` 时，`p=0.01` 只有约 `7.73%` 的组有有效相对信号，而 `p=0.5` 时约为 `99.22%`。所以增加组大小会增加比较机会，但也会消耗更多生成预算。

### GRPO 的信用分配边界

结果监督 GRPO 给同一回答的所有 token 使用同一个优势标量。这不等于每个 token 的参数梯度相同，因为每个 token 的 score、概率比和裁剪状态不同；但监督信号本身确实没有指出「究竟是哪一步导致成功或失败」。

GAE 可以通过前缀价值的变化形成不同位置的残差，时间结构更细；但这种细化来自 critic 的预测，并不自动等于真实的因果信用分配。错误的 critic 也可能给出错误的局部解释。

另外，原始 DeepSeekMath 不只讨论了结果监督，也给出了过程监督版本，把后续推理步骤的归一化奖励累加为 token 优势。所以「GRPO 必然只能使用终局奖励」也不准确。

## 十一、把三者放在一张图里

可以这样记：

```text
GAE
  作用：估计优势
  输入：奖励、价值预测、γ、λ
  关键：塑形奖励的 γλ 折扣蒙特卡洛回报

PPO
  作用：利用优势更新策略
  关键：概率比 + 裁剪/惩罚，控制旧数据上的更新幅度

GRPO
  作用：用同题多回答的组内比较构造优势
  关键：省去学习型 critic，但保留 PPO 风格更新与 KL 正则
```

它们不是同一层的三个算法。更准确的说法是：

- GAE 是优势估计器，单独不是完整训练算法。
- PPO 是策略更新方法，可以使用 GAE，也可以使用别的优势估计。
- GRPO 改变了采样结构与优势构造，同时沿用 PPO 风格的局部更新机制。

## 十二、最容易记错的几句话

- 「减去同一个常数，奖励方差不变，所以 baseline 没用。」——奖励方差不变，不代表 `score × 奖励` 的梯度方差不变。
- 「GAE 只有 λ=1 才是蒙特卡洛估计。」——对任意 λ，它都是塑形奖励的 `γλ` 折扣蒙特卡洛回报；λ=1 才进一步化成原回报减 baseline。
- 「λ 小，就只考虑短期奖励。」——价值函数仍然在预测长期回报；λ 小只是更早相信这个预测，少累积后续残差。
- 「γ 和 λ 只通过乘积出现，可以互换。」——γ 还进入 TD 残差和价值目标，二者不能互换。
- 「几何衰减能保证训练收敛。」——它改善无穷和与递推的控制，不构成神经网络训练收敛的定理。
- 「GRPO 没有 critic，所以没有 baseline。」——它使用的是同题组内奖励 baseline，用重复采样替代了一部分学习型价值模型的成本。
- 「PPO 裁剪能自动纠正错误优势。」——裁剪改变的是更新激励，不会把错误方向变成正确方向。

## 十三、如果继续往下做

如果要把这篇笔记变成可验证的实验，我会优先补三件事：

1. 固定同一批 on-policy 数据，比较 `λ=0, 0.9, 0.95, 1.0` 下的优势分布、梯度方差和策略更新幅度。
2. 单独记录 critic 的解释方差。否则很难判断性能变化来自 GAE 的 λ，还是来自价值模型本身变好或变差。
3. 把终止与截断分开记录，明确 bootstrap 和 mask 的处理；这一类实现细节足以改变实验结论。

## 参考资料

- Schulman et al., *High-Dimensional Continuous Control Using Generalized Advantage Estimation*, 2015. [arXiv:1506.02438](https://arxiv.org/abs/1506.02438)
- Schulman et al., *Proximal Policy Optimization Algorithms*, 2017. [arXiv:1707.06347](https://arxiv.org/abs/1707.06347)
- Shao et al., *DeepSeekMath: Pushing the Limits of Mathematical Reasoning in Open Language Models*, 2024. [arXiv:2402.03300](https://arxiv.org/abs/2402.03300)
- Sutton & Barto, *Reinforcement Learning: An Introduction*, 2nd ed., 2018. [MIT Press](http://incompleteideas.net/book/the-book-2nd.html)
- Greensmith, Bartlett & Baxter, *Variance Reduction Techniques for Gradient Estimates in Reinforcement Learning*, JMLR 2004. [JMLR](https://www.jmlr.org/papers/v5/greensmith04a.html)
- Ng, Harada & Russell, *Policy Invariance Under Reward Transformations: Theory and Application to Reward Shaping*, 1999. [PDF](https://ai.stanford.edu/~ang/papers/shaping-icml99.pdf)

---

这篇笔记的核心只有一句话：**GAE 不是「把 TD 和蒙特卡洛折中一下」这么模糊，它是在用价值函数改变奖励表示，再对一个可调的折扣回报做估计。** 想清楚这一点，`γ`、`λ`、baseline、PPO 和 GRPO 的位置就不容易混了。
