# David Silver / UCL强化学习讲座笔记

`2020年3月13日 10:56 AM`

Daivd Silver，Deep Mind的AlphaGo系列成果的主要贡献者；该讲座为UCL举办，共十节，资源链接如下：

* [课程主页](http://www0.cs.ucl.ac.uk/staff/d.silver/web/Teaching.html)
* [中文资源](https://github.com/datawhalechina/UCL-DeepReinforcementLearning)（讲座视频在B站搜David Silver即可。）

对讲座的总体感觉：难度适中、讲解清晰明了且具启发性，比以往接触过的任何强化学习资料都更有价值。拟把学习到的知识应用到kaggle.com上的Blackjack编程挑战中，该项目链接如下：

* [Kaggle.com Blackjack challenge](https://www.kaggle.com/kernels/fork/1315754)
* [项目讨论](https://www.kaggle.com/learn-forum/58735#latest-348767)（含其他人的多种不同解题思路，有机器学习的方法，但似乎未见强化学习的方法。）
  
## Markov Decision Process

马尔可夫决策过程，MDP，是整个强化学习讲座的基础，是解决大部分强化学习问题的基准模型。MDP包含的知识点如下：

* Markov Property，马尔可夫性，即下一状态只由当前状态决定，与过往的状态历史无关，即：$\mathbb{P}[S_{t+1}|S_t]=\mathbb{P}[S_{t+1}|S_1,\dots,S_t]$；只有符合马尔可夫性的场景才能应用MDP去建模和解决问题。
* Agent / Environment，Agent是产生动作的主体，一般可由用户完全控制其运动，Environment即与Agent交互的环境，可以是完全可知的、部分可知的或不可知的；在MDP模型中，用户指挥Agent执行动作，与Environment产生交互，发生状态转移，并从Environment中获得Reward（收益）；Agent在当前状态下如何执行动作，以获得整体最大的收益（一般来说，就是让Agent从当前状态运动到终结状态所能获得的收益总和最大），是大部分强化学习要解决的问题，即寻找最优策略$\pi^*$。
* Observation/Reward，Agent执行一个动作，获得收益，在环境中的处境也随之改变（比如位置、速度、温度等等），这些都属于observation，需要注意的是，MDP里面的状态（State）绝不等同于observation，State是某种统计性质的抽象，可以认为是对Agent来到此状态之前的过往全部历史的统计性质的综合；除此之外，很多时候MDP算法里面的Reward与Environment给予Agent的Reward并不完全一样，因为Environment可能只会在Agent到达终结状态才会给予可见的奖励，如电子游戏到本局终结时才判定玩家输赢，在此之前，并无任何显式的收益回馈，此时则要求MDP算法设计者设计出合理的Reward函数，使得Agent执行每一个动作都能获得反馈，从而能够为获得最优动作决策提供依据。
* MDP构成要素：$\mathcal{S}$（状态空间）、$\mathcal{A}$（动作空间）、$\mathcal{R}$（收益函数）、$\mathcal{P}$（状态转移矩阵）、$\pi$（策略函数）与$\gamma$（折扣因子）。其中，$\mathcal{R}$表征在当前状态$s$下执行动作$a$所获得的收益期望，即$\mathcal{R}_s^a=\mathbb{E}[R_{t+1}|S_t=s,A_t=a]$（有时候也表征为获得$R$的概率分布，即$\mathcal{R}(s,a)=\mathbb{P}[R|s,a]$）；$\mathcal{P}$表征在当前状态$s$下执行动作$a$转移到状态$s'$的概率，即$\mathcal{P}_{ss'}^a=\mathbb{P}[S_{t+1}=s'|S_t=s,A_t=a]$；$\pi$表征在当前状态$s$下执行动作$a$的概率，即$\pi(a|s)=\mathbb{P}[A_t=a|S_t=s]$，有时候$\pi$不作概率解释而是直接返回所选择的动作$a$，即$\pi(s)=a$；至于折扣因子$\gamma\in[0,1]$，引入它是基于这样的一种理念：Agent处于当前状态下，能获得的即时收益是确定的，而转移到未来的状态后所获得的收益是不确定的，因此在考虑未来收益期望的时候，需要以折扣因子来减弱后者的影响力。
* Envrionment的可知性：如果收益函数$\mathcal{R}$与状态转移矩阵$\mathcal{P}$完全可知，则这个MDP的Environment是可知的，能求出最优策略；然而大部分的现实问题都不具备这样的条件。
* Value Function：价值函数或状态价值函数，表征对从当前状态往后所有状态（直到运行到终结状态或无限远的未来）可能获得的收益总和的数学期望，以递推形式表述则可以写作$v_\pi(s)=\mathbb{E}_\pi[G_t|S_t=s]=\mathbb{E}_\pi[R_{t+1}+\gamma v_\pi(S_{t+1})|S_t=s]$，其中$G_t=R_{t+1}+\gamma R_{t+2}+\dots+=\sum_{k=0}^\infty\gamma^kR_{t+k+1}$，表征从$t$时刻往后的带折扣的收益总和，$R_{t+1}$是在状态$S_t=s$下执行某一个动作$a$所获得的即时收益，$v(S_{t+1})$是下一个状态对应的状态价值，这就是所谓的Bellman Equation。
* Action-Value Function：与状态价值函数类似，动作价值函数$q(s, a)$，表征Agent在状态$s$下，执行动作$a$之后，一直到到终结状态（或无限远的未来）所能获得的总收益的数学期望，可表示为：$q_\pi(s,a)=\mathbb{E}_\pi[G_t|S_t=s,A_t=a]=\mathbb{{E}_\pi}[R_{t+1}+\gamma q_\pi(S_{t+1},A_{t+1})|S_t=s,A_t=a]$，其中$R_{t+1}$为即时获得的收益，由于$s$和$a$是确定的，所以$R_{t+1}$是确定的。
* Value Function与Action-Value Function的递推关系（即Bellman Expectation Equation）：
  * $v_\pi(s)=\sum\limits_{a\in A}{\pi(a|s)q_\pi(s,a)}=\sum\limits_{a\in A}{\pi(a|s)(\mathcal{R}_s^a+\gamma\sum\limits_{s'\in\mathcal{S}}{\mathcal{P}_{ss'}^av_\pi(s')})}$
  * $q_\pi(s,a)=\mathcal{R}_s^a+\gamma\sum\limits_{s'\in\mathcal{S}}{\mathcal{P}_{ss'}^av_\pi(s')}=\mathcal{R}_s^a+\gamma\sum\limits_{s'\in\mathcal{S}}{\mathcal{P}_{ss'}^a\sum\limits_{a'\in\mathcal{A}}{\pi(a'|s')q_\pi(s',a')}}$
* Bellman Optimality Equation（非线性，无闭式解，需借助迭代方法求解）：
  * $v_*(s)=\max\limits_a\mathcal{R}_s^a+\gamma\sum\limits_{s'\in\mathcal{S}}{\mathcal{P}_{ss'}^av_*(s')}$
  * $q_*(s,a)=\mathcal{R}_s^a+\gamma\sum\limits_{s'\in\mathcal{S}}{\mathcal{P}_{ss'}^a\max\limits_{a'}q_*(s',a')}$

一般地，称求得状态价值函数的问题为预测问题（Prediction，某策略$\pi$下的价值函数记为$v_\pi$，最优价值函数记为$v^*$），而求得最优策略的问题为控制问题（Control，最优策略记为$\pi^*$）。

## Dynamic Programming

动态规划，DP，解决某类型的被广泛采用的方法（适用于包含所谓“最优子结构”以及可以递归分解、用递推关系表述的问题），包括某类型MDP问题。要使用DP，MDP问题需要符合下面条件：

* MDP是episodic的，即存在终结状态，Agent通过状态转移最后会到达终结状态，而不存在无限的状态转移。
* MDP是完全可知的，即$\mathcal{R}$与$\mathcal{P}$是确定的（有时候$\mathcal{R}$需要开发者选定）。
* 状态空间$\mathcal{S}$与动作空间$\mathcal{A}$的规模是合理大小的，不能庞大得超越实际算力限制。
* DP只能应用于offline的情形，即不能一边产生新数据一边求解最优策略。
* 基于Policy Iteration的DP算法：
  * 初始化策略$\pi$，一般初始化为统一随机策略（uniform random policy），即以$\frac{1}{|\mathcal{A}|}$的概率选择策略$a\in\mathcal{A}$；初始化各$v_\pi(s)$，一般为0，特别是终结状态$v_\pi(s_T)=0$；选定收益函数$\mathcal{R}$。
  * 算法通过多次迭代来让当前策略$\pi$迫近最优策略$\pi^*$，每一次迭代都包含Policy Evaluation和Policy Improvement两阶段，前一阶段更新全部状态价值$v_\pi(s)$，后一阶段调整当前策略$\pi$。
  * Policy Evaluation阶段：依循递推关系Bellman Expectation Equation为每一个$s\in\mathcal{S}$计算基于当前策略的状态价值$v_\pi(s)$，即$v_\pi(s)=\mathbb{E}[R_{t+1}+\gamma R_{t+2}+\dots|S_t=s]=\sum\limits_{a\in\mathcal{A}}\pi(a|s)(\mathcal{R}_s^a+\gamma\sum\limits_{s'\in\mathcal{S}}(\mathcal{P}_{ss'}^av_\pi(s')))$，实际应用中一般仅使用后式——对于本轮迭代来说，$v_\pi(s')$是已知，即对于第一轮，它就是初始化值（一般为$0$），对于第$t$轮，它的值已在第$t-1$轮算出，所以这是一种只需要“前向一步”的算法。
  * Policy Improvement阶段：以贪心算法更新策略，$\pi'(s)=greedy(v_\pi)=\arg\max\limits_{a\in\mathcal{A}}q_\pi(s,a)=\arg\max\limits_{a\in\mathcal{A}}\mathcal{R}_s^a+\sum\limits_{s'\in\mathcal{S}}{\mathcal{P}_{ss'}^av_\pi(s')}$。
  * 重复上述两阶段的Policy Iteration，直到收敛（达到收敛条件或指定的迭代次数），即得$\pi'\rightarrow\pi^*$。
* 基于Value Iteration的DP算法：
  * 无需指定初始策略，也没有显式的策略迭代，仅依据Bellman Optimality Equation在每一轮迭代中更新每个状态价值：$v_{k+1}(s)=\max\limits_a\mathcal{R}_s^a+\gamma\sum\limits_{s'\in\mathcal{S}}{\mathcal{P}_{ss'}^av_k(s')}$。
  * 理论上Value Iteration能达成$v\rightarrow v^*$。
  
## Monte-Carlo Method

蒙地卡罗法，MC，一种直观而有效的、依赖随机抽样的算法。与DP比较，他们的主要差别可以从状态价值$v_\pi$的计算方法的不同来区分：

* 带折扣的收益总和（total discounted reward）：$G_t=R_{t+1}+\gamma R_{t+2}+\dots+\gamma^{T-1}R_T$。
* 在DP下，指定策略$\pi$下的状态价值函数（Value Function under policy $\pi$）是$G_t$的数学期望：$v_\pi(s)=\mathbb{E}_\pi[G_t|S_t=s]$。实践中，DP一般不直接计算$G_t$，而是在Policy Evaluation阶段的每轮迭代中从$\mathcal{P}$和$\mathcal{R}$算得每一个状态的$v_\pi$，并以“前向一步”的方式以上一轮迭代的结果来更新本轮的各$v_\pi$。
* 在MC中，$v_\pi$不再是$G_t$的数学期望，而是它的经验均值（empirical mean），即让Agent在选定的$\pi$下，从初始状态$s_1$出发，与环境不断交互，直至到达终结状态$s_T$，以此产生的状态/动作/收益序列$s_1,a_1,R_2,s_2,a_2,R_3,\dots,s_T$记为一个episode，其对应的收益序列$R_2,R_3,\dots,R_T$能直接算出带折扣的收益总和$G_t$，而$v_\pi$的更新方式则是：

$$
\begin{gathered}
N(S_t)\leftarrow N(S_t)+1 &\text{for each state $S_t$ with $G_t$} \\
v(S_t)\leftarrow v(S_t)+\frac{1}{N(S_t)}(G_t-v(S_t))
\end{gathered}
$$

另一个更简单的公式也较常用（特别对于非稳态问题）：$v(S_t)\leftarrow v(S_t)+\alpha(G_t-v(S_t))$，$\alpha\in(0,1)$为学习率。MC的Policy Evaluation阶段就是用大量episode计算各$v_\pi(s)$。

从上可以看出MC需要MDP问题符合一定条件才能适用：

* MDP必须是episodic的，用于学习的episode也必须是完整的（即episode序列必须包含从初始状态到终结状态的全部信息状态、动作与收益信息）。
* $\mathcal{P}$与$\mathcal{R}$可以是未知的，也就是MC可以是model-free的；但有时候开发者需要人为设计合理的$\mathcal{R}$。
* MC只能应用于offline学习。

上面讨论的是MC的Prediction问题，而它的Control问题的求解，也是遵循基本的Policy Iteration方法：

* Model-Free On-Policy MC Control的基本要素（On-Policy即采样时使用的$\pi$与优化目标$\pi$是同一策略，与之相反的称为Off-Policy）：
  * 初始化$\pi$、各$v(s)$，确定$\mathcal{R}$。
  * Policy Evaluation阶段，以当前$\pi$进行采样，获得episode，求出对应的状态-动作组合$Q$值，即$Q=q_\pi$；重复这个采样-求值过程，直到$Q$趋于收敛。
  * Policy Improvement阶段，以$\epsilon$-greedy的方式更新$\pi$（以大概率$1-\epsilon$选择当前最优策略，小概率$\epsilon$探索随机策略），即：

$$
\pi(a|s)=
\begin{cases}
\frac{\epsilon}{|\mathcal{A}|}+1-\epsilon &\text{if }a^*=\arg\max\limits_{a\in\mathcal{A}}Q(s,a) \\
\frac{\epsilon}{|\mathcal{A}|} &\text{otherwise}
\end{cases}
$$

重复两阶段的Policy Iteration，即有$\pi\rightarrow\pi^*$。上述过程，亦可以对每一个episode都执行Policy Evaluation与Policy Improvement两阶段，而不是在Policy Evaluation阶段执行多个episode直到$Q$收敛再更新$\pi$，GLIE（Greedy in the Limit with Infinite Exploration）法即采用这种思路，其算法流程是：

* 以当前$\pi$采样第$k$个episode：$\{s_1,a_1,R_2,\dots,s_T\}\sim\pi$。
* 对该episode里面的每一个$S_t,A_t$对，执行：

$$
\begin{gathered}
N(S_t, A_t)\leftarrow N(S_t,A_t)+1 \\
Q(S_t,A_t)\leftarrow Q(S_t,A_t)+\frac{1}{N(S_t,A_t)}(G_t-Q(S_t,A_t))
\end{gathered}
$$

* 更新$\pi$：

$$
\begin{gathered}
\epsilon\leftarrow\frac{1}{k} \\
\pi\leftarrow\epsilon\text{-greedy}(Q)
\end{gathered}
$$

* 重复上述迭代过程，即得$\pi\rightarrow\pi^*$。

容易注意到，与DP法相比，DP在每一次迭代中更新每一个状态的价值（因为$\mathcal{P}$与$\mathcal{R}$已知，所以这是可以做到的），而MC仅更新episode中包含的状态的价值。

## Temporal Difference Method

时域差分法，TD。TD法对MDP的适用条件比较宽松，应用面也更广：

* TD法不要求MDP是episodic的，故可用于online学习。
* TD法也是model-free的，与MC类似。
* TD法从不完整的episode中学习（以bootstrapping的方式），本质上是让一种估计去迫近另一种估计，而不是MC那样从完整的经验数据（episode）中学习收益总和的经验均值（或者说，TD法的迭代迫近目标是有偏估计，而MC是无偏估计）。
* 从bias/variance trade-off角度比较，MC是0 bias但high variance，TD是some bias但low variance。
* 比起MC，TD法一般有较高的效率，但在某些情况下无法收敛到$v_\pi$，而且对初始值的选择敏感。

最简单的TD是只“向前看一步”的TD(0)，在Prediction问题上，TD(0)算法与MC类似，但状态价值函数的更新公式变为：
$$v(S_t)\leftarrow v(S_t)+\alpha(R_{t+1}+\gamma v(S_{t+1})-v(S_t))$$

在TD(0)的基础上，把前向观察的步数增加，即得TD($\lambda)$法，其中$\lambda\in[0,1]$，其解释如下：

* 定义n步收益（n-step return）：$G_t^{(n)}=R_{t+1}+\gamma R_{t+2}+\dots+\gamma^{n-1}R_{t+n}+\gamma^nv(S_{t+n})$。
* 则n步的TD学习公式为：$v(S_t)\leftarrow v(S_t)+\alpha(G_t^{(n)}-v(S_t))$。
* 定义$\lambda$-收益（$\lambda$-return）为：$G_t^\lambda=(1-\lambda)\sum\limits_{n=1}^\infty{\lambda^{n-1}G_t^{(n)}}$，$\lambda\in[0,1]$；$G_t^\lambda$的意义即以权重$(1-\lambda)\lambda^{n-1}$来累加全部的$G_t^{(n)}$，需注意到$\sum\limits_{n=1}^\infty{(1-\lambda)\lambda^{n-1}}=1$。
* 状态价值函数更新公式为：$v(S_t)\leftarrow v(S_t)+\alpha(G_t^\lambda-v(S_t))$。

可以看到，$\lambda=0$即TD(0)，而TD(1)等价于MC，即MC法就是前向观察直到终结状态的TD。上述的状态价值函数更新公式是TD($\lambda$)的理论公式，称为Foward-View TD，它要求拥有完整的episode去计算出全部的$G_t^{(n)}$（也就是offline学习），适用条件苛刻，所以在实践中，一般使用的是Backward-View TD，其算法流程如下：

* 为每个状态$s$定义eligibility因子$E(s)$以表征其影响力随着步数推移所产生的变化——每当状态$s$被访问到则影响力增加到最大，而当后续过程无$s$出现，则其影响力随步数增加而衰减至趋于0：

$$
\begin{gathered}
E_0(s)=0 \\
E_t(s)=\gamma \lambda E_{t-1}(s)+1(S_t=s)
\end{gathered}
$$

* 状态价值函数的更新公式：

$$
\begin{gathered}
\delta_t=R_{t+1}+\gamma v(S_{t+1})-v(S_t) \\
v(s)\leftarrow v(s)+\alpha\delta_tE_t(s)
\end{gathered}
$$

* 需要注意的是，在Backward-View TD算法中，每一步的更新不仅仅是更新当前访问的状态$s$的eligibility因子$E(s)$和价值函数$v(s)$，还要更新其余所有状态的$E$及$v$（即无论在第$t$步中有无访问到状态$s$，$E_t(s)$和$v(s)$都要进行更新），由$E$的计算方法可知，状态价值$v(s)$会随着它长时间得不到访问而趋近定值（因为增量趋于0）。
* **疑问：在更新$v$过程中，如果$S_t\ne s$（即$s$非当前状态），那么计算$\delta_t$时，$R_{t+1}$如何取值？是否简单地取0呢？考察下文的SARSA($\lambda$)算法，显然不是，而是沿用当前的$R_{t+1}$，原因不明。**

可见Backward-View TD能提供真正的online学习能力：因为它的状态价值更新发生在每一步之后，而且不要求序列（episode）是完整的。

关于TD的Control问题，著名的算法是SARSA；On-Policy Control的SARSA算法概念如下：

* 遵循常规的两阶段的Policy Iteration结构，但两个阶段作用于每一步，而不是先让Policy Evaluation阶段的Q值收敛，这类似于MC的GLIE法的思想。
* Policy Evaluation阶段：$Q(S,A)\leftarrow Q(S,A)+\alpha(R+\gamma Q(S',A')-Q(S,A))$。
* Policy Improvement阶段：$\pi=\epsilon$-greedy($Q$)。

具体算法流程：

* Initialize $Q(s,a),\forall s\in\mathcal{S},a\in\mathcal{A}(s)$, arbitrarily, and $Q(terminal-state,\cdot)=0$
* Repeat for each episode:
  * Initialize $S$
  * Choose $A$ from $S$ using policy derived from $Q$ (e.g.,$\epsilon$-greedy)
  * Repeat for each step of episode:
    * Take action $A$, observe $R$, $S'$
    * Choose $A'$ from $S'$ using policy derived from $Q$ (e.g.,$\epsilon$-greedy)
    * $Q(S,A)\leftarrow Q(S,A)+\alpha(R+\gamma Q(S',A')-Q(S,A))$
    * $S\leftarrow S'$, $A\leftarrow A'$
  * Until $S$ is terminal.

在满足特定条件的前提下，SARSA可以收敛到最优“动作-价值”函数，即$Q(s,a)\rightarrow q_*(s,a)$。

仿照Prediction问题里面的Backward-View TD($\lambda$)，有SARSA($\lambda$)算法如下：

* Initialize $Q(s,a)$ arbitrarily, for all $s\in\mathcal{S},a\in\mathcal{A}$
* Repeat for each episode:
  * E(s,a)=0, for all $s\in\mathcal{S},a\in\mathcal{A}$ (**疑问：为什么要在每个episode前把eligibility因子清零呢？**)
  * Initialize $S,A$ (**疑问：为什么是人为指定$A$初始值而不是用$\pi$去为$S$选取$A$呢？**)
  * Repeat for each step of episode:
    * Take action $A$, observe $R,S'$
    * Choose $A'$ from $S'$ using policy derived from $Q$ (e.g.,$\epsilon$-greedy)
    * $\delta\leftarrow R+\gamma Q(S',A')-Q(S,A)$
    * $E(S,A)\leftarrow E(S,A)+1$
    * For all $s\in\mathcal{S},a\in\mathcal{A(s)}$:
      * $Q(s,a)\leftarrow Q(s,a)+\alpha\delta E(s,a)$
      * $E(s,a)\leftarrow\gamma\lambda E(s,a)$
    * $S\leftarrow S',A\leftarrow A'$
  * Until $S$ is terminal.

## Off-Policy Control with Q-Learning

Off-Policy学习是相较于On-Policy而言的，它的主要特征是：采样所用的策略与目标策略（即优化目标，$\pi$）不同，采样所用策略也称为行为策略（behaviour policy），一般记为$\mu$。

Q-Learning与SARSA相似，但有几个不同点：

* 第一个$A$，即$A_t$，Q-Learning是从行为策略$\mu$获得的：$A_t\sim\mu(\cdot|S_t)$。
* 第二个$A$，即$A'$，Q-Learning从目标策略$\pi$获得：$A'\sim\pi(\cdot|S')$。
* $Q(S_t,A_t)$的更新公式是：$Q(S_t,A_t)\leftarrow Q(S_t,A_t)+\alpha(R_{t+1}+\gamma Q(S_{t+1},A')-Q(S_t,A_t))$。
* 在Policy Improvement阶段，$\mu$和$\pi$都会得到更新：

$$
\begin{gathered}
\mu(S_t)=\epsilon\text{-greedy}(Q_t,A_t) \\
\pi(S_{t+1})=\arg\max\limits_{a'}Q(S_{t+1},a'))  
\end{gathered}
$$

* 据上式，得$Q$的实际更新公式为：$Q(S,A)\leftarrow Q(S,A)+\alpha(R+\gamma \max\limits_{a'}Q(S',a')-Q(S,A))$，因此Q-Learning又称为SARSAMax。

具体算法流程如下：

* Initialize $Q(s,a),\forall s\in\mathcal{S},a\in\mathcal{A}$, arbitrarily, and $Q(terminal-state,\cdot)=0$
* Repeat for each episode:
  * Initialize $S$
  * Repeat for each step of episode:
    * Choose $A$ from $S$ using policy (behaviour policy $\mu$) derived from $Q$ (e.g.,$\epsilon$-greedy)
    * Take action $A$, observe $R,S'$
    * $Q(S,A)\leftarrow Q(S,A)+\alpha(R+\gamma \max\limits_aQ(S',a)-Q(S,A))$
    * $S\leftarrow S'$
  * Until $S$ is terminal.

## Value Function Approximation

上述的各种MDP问题的解法，实践上都属于查表法——即$v$或$Q$值在内存中以查询表的形式存在，每一个状态$s$或者状态-动作组合$\langle s,a\rangle$对应表中一个条目。这对于大规模的MDP问题，或$\mathcal{S}$、$\mathcal{A}$是连续空间的情况，查表法要么效率低下（耗费内存，且需要对每一个状态$s$都做更新），要么是根本无法实现。

价值函数近似即为解决查表法的局限而生，它通过参数化价值函数$v$或$q$，实现对大规模或连续空间MDP问题的有效求解：

$$
\begin{gathered}
\hat{v}(s,\mathbf{w})\approx v_\pi(s) \\
\hat{q}(s,a,\mathbf{w})\approx q_\pi(s,a)
\end{gathered}
$$

其中参数$\mathbf{w}$由MC或TD法学习得到。常用的近似函数有线性函数、神经网络、决策树等，一般来说，会考虑可微的函数，如线性函数与神经网络。

最常用也最简单的近似函数就是线性函数，涉及几个主要概念：

* 首先是如何表示状态$S$的问题，引入特征向量（feature vector），即把状态分解为若干个分量（特征）：

$$
\mathbf{x}(S)=
\left(
  \begin{array}{c}
  \mathbf{x}_1(S) \\
  \vdots \\
  \mathbf{x}_n(S)
  \end{array}
\right)
$$

* 把价值函数表示成状态特征的线性组合，即：

$$
\hat{v}(S,\mathbf{w})=\mathbf{x}(S)^\mathrm{T}\mathbf{w}=\sum\limits_{j=1}^n\mathbf{x}_j(S)\mathbf{w}_j
$$

* 此时（优化）目标函数则是关于$\mathbf{w}$的二次函数：

$$
J(\mathbf{w})=\mathbb{E}_\pi[(v_\pi(S)-\mathbf{x}(S)^\mathrm{T}\mathbf{w})^2]
$$

* 对目标函数应用随机梯度下降法能收敛到全局最优解，$\mathbf{w}$的更新公式是：

$$
\begin{gathered}
\nabla_\mathbf{w}\hat{v}(S,\mathbf{w})=\mathbf{x}(S) \\
\Delta\mathbf{w}=\alpha(v_\pi(S)-\hat{v}(S,\mathbf{w}))\mathbf{x}(S)
\end{gathered}
$$

上述算法属于监督学习，但实际上$v_\pi$是不可知的，这时就需要替代品，对于MC来说，替代品是$G_t$，对于TD(0)来说，是TD目标$R_{t+1}+\gamma \hat{v}(S_{t+1},\mathbf{w})$，对于TD($\lambda$)来说，是$\lambda$-收益$G_t^\lambda$。下面只列出Backward-View Linear TD($\lambda$)在Prediction问题的更新公式：

$$
\begin{gathered}
\delta_t=R_{t+1}+\gamma\hat{v}(S_{t+1},\mathbf{w})-\hat{v}(S_t,\mathbf{w}) \\
E_t=\gamma\lambda E_{t-1}+\mathbf{x}(S_t) \\
\Delta\mathbf{w}=\alpha\delta_tE_t
\end{gathered}
$$

对于Control问题，也与上述类似，下面只列出关键公式：

* 特征向量与Prediction问题不同：

$$
\mathbf{x}(S,A)=
\left(
\begin{array}{c}
\mathbf{x}_1(S,A) \\
\vdots \\
\mathbf{x}_n(S,A)
\end{array}
\right)
$$

* Backward-View Linear TD($\lambda$)的参数更新公式：

$$
\begin{gathered}
\delta_t=R_{t+1}+\gamma\hat{q}(S_{t+1},A_{t+1},\mathbf{w})-\hat{q}(S_t,A_t,\mathbf{w}) \\
E_t=\gamma\lambda E_{t-1}+\nabla_\mathbf{w}\hat{q}(S_t,A_t,\mathbf{w}) \\
\Delta\mathbf{w}=\alpha\delta_t E_t
\end{gathered}
$$

根据上述公式，可以构造出Linear SARSA($\lambda$)算法，不作赘述。

Prediction算法的收敛性比较：

|On/Off-Policy|Algorithm    |Table Lookup|Linear      |Non-Linear  |
|:-----------:|:-----------:|:----------:|:----------:|:----------:|
|On-Policy    |MC           |$\checkmark$|$\checkmark$|$\checkmark$|
|             |LSMC         |$\checkmark$|$\checkmark$|N/A         |
|             |TD(0)        |$\checkmark$|$\checkmark$|$\times$    |
|             |TD($\lambda$)|$\checkmark$|$\checkmark$|$\times$    |
|             |Gradient TD  |$\checkmark$|$\checkmark$|$\checkmark$|
|             |LSTD         |$\checkmark$|$\checkmark$|N/A         |
|Off-Policy   |MC           |$\checkmark$|$\checkmark$|$\checkmark$|
|             |TD(0)        |$\checkmark$|$\times$    |$\times$    |
|             |TD($\lambda$)|$\checkmark$|$\times$    |$\times$    |
|             |Gradient TD  |$\checkmark$|$\checkmark$|$\checkmark$|
|             |LSTD         |$\checkmark$|$\checkmark$|N/A         |

Control算法的收敛性比较：

|Algorithm          |Table Lookup|Linear        |Non-Linear     |
|:-----------------:|:----------:|:------------:|:-------------:|
|Monte-Carlo Control|$\checkmark$|($\checkmark$)|$\times$       |
|SARSA              |$\checkmark$|($\checkmark$)|$\times$       |
|Q-Learning         |$\checkmark$|$\times$      |$\times$       |
|Gradient Q-Learning|$\checkmark$|$\checkmark$  |$\times$       |
|LSPI               |$\checkmark$|($\checkmark$)|N/A            |

> ($\checkmark$)意为接近最优。

上面描述的算法都属于增量算法（incremental methods），即“采样数据-处理数据-更新参数-采样数据”这样的迭代循环，虽然较简单但并不高效，因此有另一类称为批量算法（batch methods）的方法，旨在提供高效的采样以及最优的函数拟合。这里要着重介绍的一种批量算法，是最小二乘法（Least Squares）。

Least Squares在Prediction问题的理论基础如下：

* 设近似函数$\hat{v}(s,\mathbf{w})\approx v_\pi(s)$，求解目标即求得参数$\mathbf{w}$使得$\hat{v}(s,\mathbf{w})$是最优近似函数。
* 由<状态,价值>序列组成的经验$\mathcal{D}$：$\mathcal{D}=\{\langle s_1,v_1^\pi\rangle\,\langle s_2,v_2^\pi\rangle\,\dots,\langle s_T,v_T^\pi\rangle\}$。
* 构造最小二乘法的优化目标函数：

$$
\begin{aligned}
LS(\mathbf{w})&=\sum\limits_{t=1}^T{(v_t^\pi-\hat{v}(s,\mathbf{w}))^2} \\
&=\mathbb{E}_\mathcal{D}[(v^\pi-\hat{v}(s,\mathbf{w}))^2]
\end{aligned}
$$

* 参数$\mathbf{w}$用带经验重放（experience replay）的随机梯度下降法学习：
  * 采样$\langle s,v^\pi\rangle\sim\mathcal{D}$。
  * 以随机梯度下降法更新$\mathbf{w}$：$\Delta\mathbf{w}=\alpha(v^\pi-\hat{v}(s,\mathbf{w}))\nabla_\mathbf{w}\hat{v}(s,\mathbf{w})$。
  * 重复上述步骤直到收敛，即得最小二乘法的解：$\mathbf{w}^\pi=\arg\min\limits_\mathbf{w}LS(\mathbf{w})$。
* 经验重放法也应用到DQN（Deep Q-Networks）中，算法流程是类似的：
  * 经验序列为$\mathcal{D}=\{\langle s_1,a_1,r_2,s_2\rangle\,\langle s_2,a_2,r_3,s_3\rangle\,\dots,\langle s_t,a_t,r_{t+1},s_{t+1}\rangle\}$，其中$a_t$以$\epsilon$-greedy法选择。
  * 从$\mathcal{D}$中采样一小批量样本（mini batch），记为$\mathcal{D}_i$，并固定当前的参数记为$\mathbf{w}_i^-$。
  * 以随机梯度下降法优化目标函数：$\mathcal{L}_i(\mathbf{w}_i)=\mathbb{E}_{s,a,r,s'\sim\mathcal{D}_i}[(r+\gamma\max\limits_{a'}Q(s',a';\mathbf{w}_i^-)-Q(s,a;\mathbf{w}_i))^2]$。
  * 重复上述过程直至收敛。
* 最小二乘法中的较简单常用的一种，线性最小二乘法（Linear Least Squares），即$\hat{v}(s,\mathbf{w})=\mathbf{x}(s)^\mathrm{T}\mathbf{w}$，可以用公式直接求解最优参数而无需像上面那样多次迭代至收敛：$\mathbf{w}=(\sum\limits_{t=1}^T{\mathbf{x}(s_t)\mathbf{x}(s_t)^\mathrm{T}})^{-1}\sum\limits_{t=1}^T{\mathbf{x}(s_t)v_t^\pi}$。

无论是线性还是非线性最小二乘法，真实的$v^\pi$都是不可知的，于是有不同的算法用不同的替代物去近似它：

* LSMC：$v_t^\pi\approx G_t$。
* LSTD：$v_t^\pi\approx R_{t+1}+\gamma\hat{v}(S_{t+1},\mathbf{w})$。
* LSTD($\lambda$)：$v_t^\pi\approx G_t^\lambda$。

替换近似后即能衍生出与之对应的经验重放采样迭代法及直接公式解法，不作赘述。他们的收敛性可见前表。

Least Squares在Control问题上也是沿用经典的Policy Iteration模式，其核心是以状态特征$\mathbf{x}(s,a)$的线性组合来迫近$q_\pi$：$\hat{q}(s,a,\mathbf{w})=\mathbf{x}(s,a)^\mathrm{T}\mathbf{w}\approx q_\pi(s,a)$，其中$\pi$是产生经验序列$\mathcal{D}$的策略，$\mathcal{D}=\{\langle(s_1,a_1),v_1^\pi\rangle,\langle(s_2,a_2),v_2^\pi\rangle,\dots\langle(s_T,a_T),v_T^\pi\rangle\}$，明显地，Least Squares Control必然是Off-Policy的，算法流程也与Q-Learning相似：

* 由旧策略$\pi_{old}$生成经验序列：$S_t,A_t,R_{t+1},S_{t+1}\sim\pi_{old}$。
* 以新策略$\pi_{new}$生成新动作：$A'=\pi_{new}(S_{t+1})$。
* 更新$\hat{q}(S_t,A_t,\mathbf{w})$让它迫近$R_{t+1}+\gamma\hat{q}(S_{t+1},A_{t+1},\mathbf{w})$，考虑如下的Q-Learning更新公式：

$$
\begin{gathered}
\delta=R_{t+1}+\gamma\hat{q}(S_{t+1},\pi(S_{t+1}),\mathbf{w}))-\hat{q}(S_t,A_t,\mathbf{w}) \\
\Delta\mathbf{w}=\alpha\delta\mathbf{x}(S_t,A_t)
\end{gathered}
$$

* 让$\sum\limits_{t=1}^T{\Delta\mathbf{w}}=0$，则有如下的LSTDQ算法：

$$
\begin{gathered}
0=\sum\limits_{t=1}^T{\alpha(R_{t+1}+\gamma\hat{q}(S_{t+1},\pi(S_{t+1}),\mathbf{w})-\hat{q}(S_t,A_t,\mathbf{w}))\mathbf{x}(S_t,A_t)} \\
\mathbf{w}=(\sum\limits_{t=1}^T{\mathbf{x}(S_t,A_t)(\mathbf{x}(S_t,A_t)-\gamma\mathbf{x}(S_{t+1},\pi(S_{t+1})))^\mathrm{T}})^{-1}\sum\limits_{t=1}^T{\mathbf{x}(S_t,A_t)R_{t+1}}
\end{gathered}
$$

* 最后，按简单的贪心算法去更新策略，即$\pi=greedy(q_\mathbf{w})$，完整算法流程如下：
  * function LSPI-TD($\mathcal{D},\pi_0$)
    * $\pi'\leftarrow\pi_0$
    * repeat
      * $\pi\leftarrow \pi'$
      * $Q\leftarrow LSTDQ(\pi,\mathcal{D})$
      * for all $s\in\mathcal{S}$ do
        * $\pi'(s)\leftarrow\arg\max\limits_{a\in\mathcal{A}}Q(s,a)$
      * end for
    * until($\pi\approx\pi'$)
    * return $\pi$
  * end function

LSPI的收敛性，见前表。

## Policy Gradient

上面提到的各种算法，无论是常规的MC、TD还是参数化后的用MC、TD算法的函数迫近法，在学习最优策略的时候都是从$v$或$Q$值中通过greedy（包括$\epsilon$-greedy）方式来学得，这些可以归类为Value-Based RL。试想如果抛开价值函数$v$和$Q$，直接参数化策略本身，即：$\pi_\theta(s,a)=\mathbb{P}[a|s,\theta]$，会如何呢？

上述的方法称为Policy-Based RL，有如下特点：

* 更佳的收敛性。
* 能有效求解高维度或连续动作空间的问题。
* 能学习出随机策略（举例：某些问题里面常见多个实质上不同但又无法区分的状态或者<状态-动作>组合，导致它们对应的特征向量$\mathbf{x}$以及价值$v$或$Q$会无法区分，那么对于Value-Based RL来说，这种情况下由贪心算法学得的策略很有可能出现“死胡同里兜圈”的问题，这是由此类策略的“近乎确定性（near-deterministic）”决定的，解决办法就是引入随机性，如使用Policy-Based RL方法）。
* 可能只能收敛到局部最优，而非全局最优。
* 效率不太高，而且方差较大。

学习参数化最优策略需确定优化目标函数，一般是下面几种：

* 对于episodic环境来说，采用初始值，即：$J_1(\theta)=v^{\pi_\theta}(s_1)=\mathbb{E}_{\pi_\theta}[v_1]$。
* 对于连续的环境，则使用平均价值（average value）：

$$
\begin{gathered}
J_{avV}(\theta)=\sum\limits_s{d^{\pi_\theta}(s)v^{\pi_\theta}(s)} \\
\textit{或每步的平均收益（average reward）} \\
J_{avR}(\theta)=\sum\limits_s{d^{\pi_\theta}(s)\sum\limits_a{\pi_\theta(s,a)\mathcal{R}_s^a}}
\end{gathered}
$$

> 其中$d^{\pi_\theta}$是关于$\pi_\theta$的马尔科夫链的静态分布。

由上述目标函数的定义可知，优化策略参数$\theta$意即：$\theta=\arg\max\limits_{\theta'}J(\theta')$。这里介绍的优化方法是有限差分策略梯度法（Finite Difference Policy Gradient），解释如下：

* 通过梯度上升法寻找能局部最大化$J(\theta)$的参数$\theta$，其中策略参数的更新增量为：$\Delta\theta=\alpha\nabla_\theta J(\theta)$。
* $\nabla_\theta J(\theta)$即所谓的策略梯度，Policy Gradient：

$$
\nabla_\theta J(\theta)=
\left(
\begin{array}{c}
\frac{\partial J(\theta)}{\partial\theta_1} \\
\vdots \\
\frac{\partial J(\theta)}{\partial\theta_n}
\end{array}
\right)
$$

* 设参数$\theta$维数为$n$，则对每一维$k\in[1,n]$，计算：

$$
\frac{\partial J(\theta)}{\partial\theta_k}\approx\frac{J(\theta+\epsilon u_k)-J(\theta)}{\epsilon}
$$

> $u_k$为$n$维向量，其中第$k$分量为$1$，其余分量为$0$。

由上式可知，该方法适合任意类型的策略，包括不可微分的策略。但对于可微分的策略$\pi_\theta(s,a)$，有下面的策略梯度定理（Policy Gradient Theorem）：

$$
\nabla_\theta J(\theta)=\mathbb{E}_{\pi_\theta}[\nabla_\theta\log\pi_\theta(s,a)Q^{\pi_\theta}(s,a)]
$$

> 其中$J$可以是$J_1$、$J_{av}R$或$\frac{1}{1-\gamma}J_{avV}$。
> $\nabla_\theta\log\pi_\theta(s,a)$称为Score Function，来自恒等式：$\nabla_\theta\pi_\theta(s,a)=\pi_\theta(s,a)\frac{\nabla_\theta\pi_\theta(s,a)}{\pi_\theta(s,a)}=\pi_\theta(s,a)\nabla_\theta\log\pi_\theta(s,a)$。
> 当策略是Softmax型的，即：$\pi_\theta(s,a)\propto e^{\mathbf{x}(s,a)^\mathrm{T}\theta}$，则Score Function为：
> $\nabla_\theta\log\pi_\theta(s,a)=\mathbf{x}(s,a)-\mathbb{E}_{\pi_\theta}[\mathbf{x}(s,\cdot)]$；
> 当策略是高斯型的，即：$\pi_\theta(s,a)\propto\mathcal{N}(\mu(s),\sigma^2)$，其中均值$\mu(s)=\mathbf{x}(s)^\mathrm{T}\theta$，则Score Function为：
> $\nabla_\theta\log\pi_\theta(s,a)=\frac{(a-\mu(s))\mathbf{x}(s)}{\sigma^2}$。

由上述定理，以$v_t$来作为对$Q^{\pi_\theta}(s_t,a_t)$的无偏采样，使用随机梯度上升法，有如下的Monte-Carlo Policy Gradient (REINFORCE)：

* function REINFORCE
  * Initialise $\theta$ arbitrarily
  * for each episode $\{s_1,a_1,r_2,\dots,s_{T-1},a_{T-1},r_T\}\sim\pi_\theta$ do
    * for $t=1$ to $T-1$ do
      * $\theta\leftarrow\theta+\alpha\nabla_\theta\log\pi_\theta(s_t,a_t)v_t$
    * end for
  * end for
  * return $\theta$
* end function

鉴于上述基于蒙地卡罗采样的方法有较大的方差，需要把$Q^{\pi_\theta}$也参数化以解决问题，即ACtor-Critic Policy Gradient：

* Critic，更新动作价值函数$Q^{\pi_\theta}$的参数$\mathbf{w}$。
* Actor，依据Critic提供的信息，更新策略参数$\theta$。
* Actor-Critic的核心思想体现为下面的公式：

$$
\begin{gathered}
\nabla_\theta J(\theta)\approx\mathbb{E}_{\pi_\theta}[\nabla_\theta\log\pi_\theta(s,a)Q_\mathbf{w}(s,a)] \\
\Delta\theta=\alpha\nabla_\theta\log\pi_\theta(s,a)Q_\mathbf{w}(s,a)
\end{gathered}
$$

Actor-Critic的一种简单基于动作价值Critic的算法如下：

* 动作价值函数使用线性近似：$Q_\mathbf{w}(s,a)=\mathbf{x}(s,a)^\mathrm{T}\mathbf{w}$。
* Critic以线性TD(0)算法更新参数$\mathbf{w}$。
* Actor以策略梯度算法更新参数$\theta$。
* function QAC
  * Initialise $s$, $\theta$
  * Sample $a\sim\pi_\theta$
  * for each step do
    * Sample reward $r=\mathcal{R}_s^a$; sample transition $s'\sim\mathcal{P}_s^a$
    * Sample action $a'\sim\pi_\theta(s',a')$
    * $\delta=r+\gamma Q_\mathbf{w}(s',a')-Q_\mathbf{w}(s,a)$
    * $\theta=\theta+\alpha\nabla_\theta\log\pi_\theta(s,a)Q_\mathbf{w}(s,a)$
    * $\mathbf{w}\leftarrow\mathbf{w}+\beta\delta\mathbf{x}(s,a)$
    * $a\leftarrow a',s\leftarrow s'$
  * end for
* end function

实践中Actor-Critic还有多种不同的策略梯度法变体，不一一详述，仅列出策略梯度公式如下：

* $\nabla_\theta J(\theta)=\mathbb{E}_{\pi_\theta}[\nabla_\theta\log\pi_\theta(s,a)v_t]$

> 即前文所述的REINFORCE算法。

* $\nabla_\theta J(\theta)=\mathbb{E}_{\pi_\theta}[\nabla_\theta\log\pi_\theta(s,a)Q_\mathbf{w}(s,a)]$

> Q Actor-Critic，即前述的QAC算法。

* $\nabla_\theta J(\theta)=\mathbb{E}_{\pi_\theta}[\nabla_\theta\log\pi_\theta(s,a)A_{\mathbf{w,v}}(s,a)]$

> Advantage Actor-Critic，其中$A^{\pi_\theta}(s,a)=Q^{\pi_\theta}(s,a)-v^{\pi_\theta}(s)$或$A_{\mathbf{w,v}}(s,a)=Q_\mathbf{w}(s,a)-v_\mathbf{v}(s)$，称为Advantage Function，此法需要在Critic阶段使用两组参数$\mathbf{w}$和$\mathbf{v}$。

* $\nabla_\theta J(\theta)=\mathbb{E}_{\pi_\theta}[\nabla_\theta\log\pi_\theta(s,a)\delta]$

> TD Actor-Critic，其中$\delta_\mathbf{v}=r+\gamma v_\mathbf{v}(s')-v_\mathbf{v}(s)$，对比Advantage Actor-Critic，此法在Critic阶段仅需一组参数$\mathbf{v}$。

* $\nabla_\theta J(\theta)=\mathbb{E}_{\pi_\theta}[\nabla_\theta\log\pi_\theta(s,a)\delta e]$

> TD($\lambda$) Actor-Critic，其中$\delta_\mathbf{v}=r_{t+1}+\gamma v_\mathbf{v}(s_{t+1})-v_\mathbf{v}(s_t);e_{t+1}=\lambda e_t+\nabla_\theta\log\pi_\theta(s,a);\Delta\theta=\alpha\delta_\mathbf{v}e_t$，此法可用于不完整的序列，即Online学习。

* $\nabla_\theta J(\theta)=G_\theta\mathbf{w}$

> Natural Actor-Critic，其中$G_\theta=\mathbb{E}_{\pi_\theta}[\nabla_\theta\log\pi_\theta(s,a)\nabla_\theta\log\pi_\theta(s,a)^\mathrm{T}]$，称为Fisher Information matrix。

理论上上述各法均能引出随机梯度上升算法，其中Critic阶段使用MC或TD式的Policy Gradient法去估算$Q^\pi(s,a)$、$A^\pi(s,a)$或$v^\pi(s)$。

## Model-Based RL

上面两节先后叙述了参数化（状态或动作）价值函数以及参数化策略的思路，它们都属于Model-Free RL。除此之外，还有一类算法，着眼于近似Model本身，即参数化Model的核心组成部分状态转移矩阵$\mathcal{P}$和收益函数$\mathcal{R}$，在此基础上，用规划（Plan）而不是学习（Learn）的方法求出价值函数或策略，这称之为Model-Based RL。

Model-Based RL基于下面的前提与假设：

* MDP $<\mathcal{S,A,P,R}>$由$\eta$参数化。

* 状态空间$\mathcal{S}$和动作空间$\mathcal{A}$已知。

* Model $<\mathcal{P_\eta,R_\eta}>$表示$\mathcal{P_\eta}\approx\mathcal{P}，\mathcal{R_\eta}\approx\mathcal{R}$，且有：

$$
\begin{gathered}
S_{t+1}\sim\mathcal{P_\eta}(S_{t+1}|S_t,A_t) \\
R_{t+1}=\mathcal{R_\eta}(R_{t+1}|S_t,A_t)
\end{gathered}
$$

* 假设状态转移与收益之间满足条件独立性，即：$\mathbb{P}[S_{t+1},R_{t+1}|S_t,A_t]=\mathbb{P}[S_{t+1}|S_t,A_t]\mathbb{P}[R_{t+1}|S_t,A_t]$。

如果把从真实环境中采样（sampled from environment, true MDP）称之为真实经验（real experience），而从model（approximate MDP）中采样则称之为模拟经验（simulated experience），介绍一种集成了从真实经验与模拟经验中学习价值函数和策略函数的算法框架，Dyna，它的特点是：

* 从真实经验中学习一个model（即学习$\mathcal{P_\eta、R_\eta}$的参数$\eta$）。

* 从真实经验以及模拟经验中学习和规划价值函数以及策略。

Dyna-Q算法：

* Initialize $Q(s,a)$ and $Model(s,a)$ for all $s\in\mathcal{S}$ and $a\in\mathcal{A}$.
* Do forever:
  * $S\leftarrow\text{current (nonterminal) state}$
  * $A\leftarrow\epsilon\text{-greedy}(S,Q)$
  * Execute action $A$; observe resultant reward, $R$, and state, $S'$
  * $Q(S,A)\leftarrow Q(S,A)+\alpha[R+\gamma\max\limits_a Q(S',a)-Q(S,A)]$
  * $Model(S,A)\leftarrow R,S'$ (assuming deterministic environment)
  * Repeat $n$ times:
    * $S\leftarrow\text{random previously observed state}$
    * $A\leftarrow\text{random action previously taken in S}$
    * $R,S'\leftarrow Model(S,A)$
    * $Q(S,A)\leftarrow Q(S,A)+\alpha[R+\gamma\max\limits_a Q(S',a)-Q(S,A)]$

另一类从模拟经验中学习的算法是搜索算法，现介绍应用于AlphaGo等项目的MCTS（Monte-Carlo Tree Search）算法（AlphaGo具体实现较下述流程复杂很多，下文仅给出一个简化的概念性的版本，以达到描述MCTS基本特点的目的）：

* 初始化model $\mathcal{M_v}$、树策略（tree policy）$\pi$（一般为$\epsilon\text{-greedy}(Q)$，或UCT，Upper Confidence Bounds for Tree，用于树内搜索，随着学习而得到更新）、默认策略（default policy）$\pi'$（一般为随机策略，固定不变，用于树外搜索）、初始状态$s_0$与搜索树：搜索树结点包含状态与对应之价值，如$Q(s,a)$值，连接上级结点与下级结点的边可表示从前一状态$s$到达下一状态$s'$所执行的动作$a$；容易知道搜索树的初始状态只有$s_0$对应的根结点。

* 从根结点开始以树策略向下搜索到达某叶子结点（树内搜索，in-tree），以该叶子结点对应的状态为当前状态$s_t$（第一次搜索时由于树中只有根结点，所以有$s_t\leftarrow s_0$）。

* 若$s_t$非终结状态，则从$s_t$开始，以默认策略$\pi'$和model $\mathcal{M}$生成$K$个模拟经验序列（树外搜索，out-of-tree），即：

$$
\{s_t,A_t^k,R_{t+1}^k,S_{t+1}^k,\dots,S_T^k\}_{k=1}^K\sim\mathcal{M_v,\pi}
$$

* 从这K个episode中以蒙地卡罗法计算里面包含的各$Q(s,a)$值（即取G的均值）：

$$
Q(s,a)=\frac{1}{N(s,a)}\sum\limits_{k=1}^K{\sum\limits_{u=t}^T{1(S_u,A_u=s,a)G_u}}
$$

* 为当前状态$s_t$以贪婪法选择对应的真实动作$a_t$：

$$
a_t=\arg\max\limits_{a\in\mathcal{A}}Q(s_t,a)
$$

* 把$s_t$的后续状态加入搜索树——这过程称为搜索树的扩展（expand），有多种方式把$s_t$的一个或多个后续状态添加进搜索树，讲座中无具体讲解。

* 反向价值更新：从$s_t$开始，往上逐级上溯，并更新沿途的结点信息（结点访问次数，价值，等），不同的应用场景对结点价值及价值更新方式有不同的设计，讲座中无具体讲解。

* 重复上述步骤直到停止条件达到：从根结点开始以树策略搜索至叶子结点，若叶子结点非终结状态，则以默认策略进行$K$轮模拟，以蒙地卡罗法从$K$轮episode中更新状态价值，把叶子结点的后续结点添加入搜索树，并反向更新各级沿途结点的价值。

## Exploitation and Exploration

选择已知的最优选项（exploitation）还是从其它可能的选项中发掘潜力（exploration），是各种RL算法中不可避免的问题，有时候对于提升RL效率和结果有重大意义。原来上来说，一般有如下几种路线：

* 简单探索法：加入噪声的贪心算法，即$\epsilon$-greedy。

* 乐观初始化法：把各状态价值初始化成最佳值，然后在学习过程中不断修正。

* 偏好不确定性：选择能提供更大的价值不确定性的动作。

* 概率匹配：让选择某动作的概率与该动作能带来的收益概率匹配。

* 基于信息的状态搜索：根据信息价值去指导前向状态搜索。

研究这类问题的基本模型是K-armed bandit（有K个摇臂的老虎机），要素定义如下：

* K-armed bandit model $<\mathcal{A,R}>$, which $\mathcal{A}$ is known set of $K$ actions ("arms").

* $\mathcal{R}^a=\mathbb{P}[r|a]$ is an unknown probability distribution over rewards.

* At each step $t$ the agent selects an action $a_t\in\mathcal{A}$.

* The environment ("bandit") generates a reward $r_t\sim\mathcal{R^{a_t}}$.

* The goal is to maximise cumulative reward $\sum\limits_{\tau=1}^t{r_\tau}$.

另定义概率regret，表征每一步的机会损失（opportunity loss），即：

$$
\begin{gathered}
Q(a)=\mathbb{E}[r|a] \\
V^*=Q(a^*)=\max\limits_{a\in\mathcal{A}}Q(a) \\
l_t=\mathbb{E}[V^*-Q(a_t)] \\
L_t=\mathbb{E}[\sum\limits_{\tau=1}^t{V^*-Q(a_\tau)}]=\sum\limits_{a\in\mathcal{A}}{\mathbb{E}[N_t(a)](V^*-Q(a))}=\sum\limits_{a\in\mathcal{A}}{\mathbb{E}[N_t(a)]\Delta_a}
\end{gathered}
$$

这里的$Q(a)$为$a$的动作价值（action-value），regret $l_t$为第$t$步的一步机会损失，而$L_t$为到第$t$步为止的累积总regret。易知最大化累积reward等价于最小化总regret，好的算法即让大的gaps（即$\Delta_a$）对应的计数$N_t(a)$尽量小，但，$\Delta_a$是不可知的。

这里仅介绍UCB（Upper Confidence Bounds）方法使得总regret最小。UCB算法基于下面的不等式（由Hoeffding's Inequality导出）：

$$
\mathbb{P}[Q(a)>\hat{Q}_t(a)+U_t(a)]\le e^{-2N_t(a)U_t(a)^2} \\
U_t(a)=\sqrt\frac{2\log t}{N_t(a)}
$$

UCB算法能使总regret达到对数渐近性（作为对比，$\epsilon$-greedy等算法仅能达到线性渐近性），即：

$$
\lim_{t\to\infty}L_t\le8\log t\sum\limits_{a|\Delta_a\gt0}{\Delta_a}
$$

其中较简单的UCB1算法即为：

$$
a_t=\arg\max\limits_{a\in\mathcal{A}}Q(a)+\sqrt\frac{2\log t}{N_t(a)}
$$

若把UCB算法与动作价值函数的线性回归相结合，则有如下的Linear UCB算法：

$$
Q(s,a)=\mathbb{E}[r|s,a] \\
Q_\theta(s,a)=\phi(s,a)^\mathrm{T}\theta\approx Q(s,a) \ \ （\phi可以看做是<s,a>的特征函数） \\
A_t=\sum\limits_{\tau=1}^t{\phi(s_\tau,a_\tau)\phi(s_\tau,a_\tau)^\mathrm{T}} \\
b_t=\sum\limits_{\tau=1}^t{\phi(s_\tau,a_\tau)r_\tau} \\
\theta_t=A_t^{-1}b_t \\
在最小二乘法回归中，Q_\theta为估算出的Q均值，而参数\theta的协方差是A^{-1}，则Q的方差\sigma^2由下式给出： \\
\sigma_\theta^2(s,a)=\phi(s,a)^\mathrm{T}A^{-1}\phi(s,a) \\
根据UCB原理，Q的置信上界是均值加上标准差（乘以一个常数c）： \\
Q_\theta(s,a)+c\sqrt{\phi(s,a)^\mathrm{T}A^{-1}\phi(s,a)} \\
最后得出动作选择策略为： \\
a_t=\arg\max\limits_{a\in\mathcal{A}}Q_\theta(s,a)+c\sqrt{\phi(s,a)^\mathrm{T}A^{-1}\phi(s,a)}
$$

## Minimax

Minimax算法用于二人零和博弈的最优策略搜索，即：

* 博弈双方交替采取行动让游戏进行下去，直到一方胜出。

* $R^1+R^2=0$，这里规定玩家1的回报为正。

* 定义联合策略$\pi=<\pi^1,\pi^2>$，且状态价值函数为：$v_\pi(s)=\mathbb{E}_\pi[G_t|S_t=s]$。

* Minimax价值函数在最小化玩家2的期望收益的同时，最大化玩家1的期望收益，即：$v_*(s)=\max\limits_{\pi^1}\min\limits_{\pi^2}v_\pi(s)$。

* Minimax策略即能达至Minimax状态价值的联合策略$\pi=<\pi^1,\pi^2>$；Minimax策略是纳什均衡的。

理论上Minimax状态价值可由深度优先搜索树寻得，但对实际应用场景来说这样计算规模过于庞大，并不现实，常用的方法是以二进制特征向量线性参数化价值函数，即：

$$
v(s,\mathbf{w})=\mathbf{x}(s)\mathbf{w}\approx v_*(s)，\ \ \mathbf{x}是二进制特征函数，即把输入状态s映射为只包含[0,1]元素的向量\mathbf{x}(s)。
$$

借此可计算出每一个当前局面的价值；为控制搜索树规模，一般只搜索到某一层深度而不是直到游戏结束，与此同时，使用alpha-beta搜索算法可以显著减少不必要的搜索量。

一般地$\mathbf{x}$的设计由先验知识指导，而参数$\mathbf{w}$可以是利用先验知识人为设计，也可以通过强化学习的手段获得，后者即所谓的Self-Play Reinforcement Learning，下面着重介绍它的概念与方法。

以Value-based RL应用于self-play，则有:

* MC：$\Delta\mathbf{w}=\alpha(G_t-v(S_t,\mathbf{w}))\nabla_\mathbf{w}v(S_t,\mathbf{w})$。

* TD(0)：$\Delta\mathbf{w}=\alpha(v(S_{t+1},\mathbf{w})-v(S_t,\mathbf{w}))\nabla_\mathbf{w}v(S_t,\mathbf{w})$。

* TD($\lambda$)：$\Delta\mathbf{w}=\alpha(G_t^\lambda-v(S_t,\mathbf{w}))\nabla_\mathbf{w}v(S_t,\mathbf{w})$。

对于游戏来说，给定$<s,a>$则后继状态$succ(s,a)$就是确定的，即有$q_*(s,a)=v_*(succ(s,a))$，则可推导出最优策略为：

$$
A_t=\arg\max\limits_a v_*(succ(S_t,a))，\ \ 对于玩家1 \\
A_t=\arg\min\limits_a v_*(succ(S_t,a))，\ \ 对于玩家2
$$
