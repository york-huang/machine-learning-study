# RL Exam Part B

Official links:

* [Questions](https://www.davidsilver.uk/wp-content/uploads/2020/03/exam-rl-questions.pdf)
* [Answers](https://www.davidsilver.uk/wp-content/uploads/2020/03/exam-rl-answers.pdf)

## 1. Given a known MDP, answer several questions

> Define the state-value $V^\pi(s)$.

$$
\begin{aligned}
V^\pi(s)&=\mathbb{E_\pi}[G_t|S_t=s],\ \ which\ G_t\ is,\\
G_t&=R_{t+1}+\gamma R_{t+2}+\ldots=\sum\limits_{k=0}^\infty{\gamma^kR_{t+k+1}} \\
\end{aligned}
$$

> Bellman expectation equation for state-value functions.

$$
\begin{aligned}
V^\pi(s)&=\mathbb{E_\pi}[R_{t+1}+\gamma V^\pi(S_{t+1})|S_t=s]=\sum\limits_{a\in\mathcal{A}}{\pi(a|s)(\mathcal{R}_s^a+\gamma\sum\limits_{s'\in\mathcal{S}}{\mathcal{P}_{ss'}^aV^\pi(s')})} \\\\
&\textit{So we have the following Bellman expectation equations,} \\\\
V^\pi(A)&=\pi(ab|A)(-8+0.5V^\pi(B)) \\
V^\pi(B)&=\pi(ba|B)(2+0.5V^\pi(A))+\pi(bc|B)(-2+0.5V^\pi(C)) \\
V^\pi(C)&=\pi(ca|C)(4+0.5(\frac{1}{4}V^\pi(A)+\frac{3}{4}V^\pi(C)))+\pi(cb|C)(8+0.5V^\pi(B))
\end{aligned}
$$

> Given uniform random policy $\pi_1(s,a)$ and initial state value of $V_1(A)=V_1(B)=V_1(C)=2$, apply 1 synchronous iteration of policy evaluation to compute a new value function $V_2(s)$.

$$
\begin{aligned}
V_2^\pi(A)&=-8+0.5*2=-7 \\
V_2^\pi(B)&=0.5(2+0.5*2)+0.5(-2+0.5*2)=1 \\
V_2^\pi(C)&=0.5(4+0.5(0.5+1.5))+0.5(8+1)=7
\end{aligned}
$$

> Apply 1 iteration of greedy policy improvement to compute a new deterministic policy $\pi_2(s)$.

$$
\begin{aligned}
\pi_2(ab|A)&=1 \\
\pi_2(ba|B)&=0,&\pi_2(bc|B)=1 \\
\pi_2(ca|C)&=0,&\pi_2(cb|C)=1
\end{aligned}
$$

> Prove that if $\pi'=greedy(V^\pi)$ then there must be $V^{\pi'}(s)\ge V^\pi(s)$ for all $s$.

$$
\begin{aligned}
&\textit{According to Bellman expectation equation, we have this } V^{\pi'}, \\\\
V^{\pi'}(s)&=\mathcal{R}_s^{a'}+\gamma\sum\limits_{s'\in\mathcal{S}}{\mathcal{P}_{ss'}^{a'}V^\pi(s')} \\
a'&=\arg\max_{a'\in\mathcal{A}}\mathcal{R}_s^{a'}+\gamma\sum\limits_{s'\in\mathcal{S}}{\mathcal{P}_{ss'}^{a'}V^\pi(s')} \\\\
&\textit{and this }V^\pi, \\\\
V^\pi(s)&=\sum\limits_{a\in\mathcal{A}}{\pi(a|s)(\mathcal{R}_s^a+\gamma\sum\limits_{s'\in\mathcal{S}}{\mathcal{P}_{ss'}^aV^\pi(s')})} \\
&\le\sum\limits_{a\in\mathcal{A}}{\pi(a|s)(\mathcal{R}_s^{a'}+\gamma\sum\limits_{s'\in\mathcal{S}}{\mathcal{P}_{ss'}^{a'}V^\pi(s')})} \\
&\le\sum\limits_{a\in\mathcal{A}}{\pi(a|s)V^{\pi'}(s)} \\
&=V^{\pi'}(s)\sum\limits_{a\in\mathcal{A}}{\pi(a|s)}=V^{\pi'}(s) \\\\
&\textit{It's proved that }V^{\pi'}(s)\ge V^\pi(s).\ \textit{Further more, if }V^{\pi'}(s)=V^\pi(s)\textit{ for all }s\textit{, then we have,} \\\\
V^{\pi'}(s)&=\max_{a\in\mathcal{A}}\mathcal{R}_s^{a}+\gamma\sum\limits_{s'\in\mathcal{S}}{\mathcal{P}_{ss'}^aV^{\pi'}(s')}=\max_\pi V^\pi(s) \\\\
&\textit{which alreday fulfills the Bellman optimality equation that we can conclude  }\pi'\textit{ is the optimal policy.}\\\\
&\textit{From another angle, }V^\pi=V^{\pi'}\textit{ implies }\sum\limits_{a\in\mathcal{A}}{\pi(a|s)(\mathcal{R}_s^a+\gamma\sum\limits_{s'\in\mathcal{S}}{\mathcal{P}_{ss'}^aV^\pi(s')})}=\max_{a\in\mathcal{A}}\mathcal{R}_s^a+\gamma\sum\limits_{s'\in\mathcal{S}}{\mathcal{P}_{ss'}^aV^\pi(s')}\textit{ from which we can conclude that} \\\\
\pi(a|s)&=
\begin{cases}
1 &\text{if }a=\arg\max\limits_{a\in\mathcal{A}}Q(s,a) \\
0 &\text{otherwise}
\end{cases} \\\\
&\textit{we can also conclude }\pi(s)=\pi'(s)\textit{ is the optimal policy.}
\end{aligned}
$$

> Define optimal state-value function V^*(s) for an MDP.

$$
V^*(s)=\max_\pi V^\pi(s)=\max_{a\in\mathcal{A}}\mathcal{R}_s^a+\gamma\sum\limits_{s'\in\mathcal{S}}{\mathcal{P}_{ss'}^aV^*(s')}
$$

> Bellman optimality equation for state-value functions.

$$
\begin{aligned}
V^*(A)&=-8+0.5V^*(B) \\
V^*(B)&=\max(2+0.5V^*(A), -2+0.5V^*(C)) \\
V^*(C)&=\max(4+0.5(\frac{1}{4}V^*(A)+\frac{3}{4}V^*(C)), 8+0.5V^*(B))
\end{aligned}
$$

> Apply 1 synchronous iteration of value iteration to compute a new value function $V_2(s)$, given $V_1(A)=V_1(B)=V_1(C)=2$

$$
\begin{aligned}
V_2(A)&=-7 \\
V_2(B)&=3 \\
V_2(C)&=9
\end{aligned}
$$

> Is $V_2(s) optimal?$

No, because $V_2(s)$ doesn't fulfill Bellman optimality equation, i.e., $V_2(s)\ne\max\limits_{a\in\mathcal{A}}\mathcal{R}_s^a+\gamma\sum\limits_{s'\in\mathcal{S}}{\mathcal{P}_{ss'}^aV_2(s')}$.

## 2. Given a undiscounted MRP with 2 states A and B, without knowing the $\mathcal{P}$ and $\mathcal{R}$, but 2 episodes are observed, answer several questions

> Using first-visit Monte-Carlo evaluation to estimate the state-value function $V(A), V(B)$.

$$
\begin{aligned}
V(A)&=\frac{2+0}{2}=1 \\
V(B)&=\frac{-3-2}{2}=-\frac{5}{2}
\end{aligned}
$$

> Repeat the previous evaluation but using every-visit Monte-Carlo.

$$
\begin{aligned}
V(A)&=\frac{2-1+1+0}{4}=\frac{1}{2} \\
V(B)&=\frac{-3-3-2-3}{4}=-\frac{11}{4}
\end{aligned}
$$

> Draw a diagram of the MRP that best explains these 2 episodes, also show the reward and transition probabilities on the diagram.

$$
\mathcal{P}=
\begin{pmatrix}
0.25 & 0.75 & 0.0 \\
0.5 & 0.0 & 0.5 \\
0.0 & 0.0 & 0.0
\end{pmatrix},\ \textit{transition matrix of 3 states: A, B and terminate.} \\
\mathcal{R}(A)=3 \\
\mathcal{R}(B)=-3
$$

> Define Bellman equation for a MRP.

$$
\begin{aligned}
V(s)&=\mathbb{E}[G_t|S_t=s] \\
&=\mathbb{E}[R_{t+1}+\gamma V(S_{t+1})|S_t=s] \\
&=\mathcal{R}_s+\gamma\sum\limits_{s'\in\mathcal{S}}{\mathcal{P}_{ss'}V(s')}
\end{aligned}
$$

> Solve the Bellman equation directly to give the true state-value function $V(A)$ and $V(B)$.

$$
\begin{aligned}
\mathcal{V}&=(I-\gamma\mathcal{P})^{-1}\mathcal{R} \\
&=
\begin{pmatrix}
0.75 & -0.75 & 0.0 \\
-0.5 & 1.0 & -0.5 \\
0.0 & 0.0 & 1.0
\end{pmatrix}^{-1}
\begin{pmatrix}
3 \\ -3 \\ 0
\end{pmatrix} \\
&=
\begin{pmatrix}
2 \\ -2 \\ 0
\end{pmatrix}
\end{aligned}
$$

> What value function would batch TD(0) find?

$$
\begin{aligned}
&\textit{Batch TD(0) converges to solution of max likelihood Markov model, i.e.,} \\\\
\hat{\mathcal{P}}_{ss'}&=\frac{1}{N(s)}\sum\limits_{k=1}^K{\sum\limits_{t=1}^{T_k}{\mathbf{1}(s_t^k,s_{t+1}^k=s,s')}} \\
\hat{\mathcal{R}}_s&=\frac{1}{N(s)}\sum\limits_{k=1}^K{\sum\limits_{t=1}^{T_k}{\mathbf{1}(s_t^k=s)r_t^k}} \\\\
&\textit{so we reach the results as we've seen in the answers to question 3 and 5.}
\end{aligned}
$$

> What value function would batch TD(1) find, using accumulating eligibility traces?

Batch TD(1) with eligibility traces is also known as backward TD(1) with offline update, which is exactly equivalent to every-visit Monte-Carlo, so the value function is the same as what we got in the answer to question 2.

> What value function would LSTD(0) find?

LSTD(0) finds the parameter $\mathbf{w}$ that fits the batch data with TD(0) as the target best, which is exactly equivalent to batch TD(0), we've already seen it in question 6, so the result is the same.

## 3. A rat involved in an undiscounted experiment sees lights and hears bells in 1 experience episode before reaching the terminal and receives food. Answer several questions as below

> Write down the sequence of feature vectors corresponding to the given episode.

$$
\begin{pmatrix}
1 \\ 0
\end{pmatrix}+0\to
\begin{pmatrix}
0 \\ 1
\end{pmatrix}+0\to
\begin{pmatrix}
1 \\ 1
\end{pmatrix}+1\to terminate
$$

> Approximate the state-value function by a linear combination: $b\cdot bell(s)+l\cdot light(s)$. If $b=2$ and $l=-2$ then write down the sequence of approximate values corresponding to this episode.

$$
2\to-2\to0\to terminate
$$

> Define the $\lambda$-return $v_t^\lambda$.

$$
\begin{aligned}
v_t^\lambda&=(1-\lambda)\sum\limits_{n=1}^\infty{\lambda^{n-1}G_t^{(n)}} \\
&=(1-\lambda)\sum\limits_{n=1}^\infty{\lambda^{n-1}(R_{t+1}+\gamma R_{t+2}+\dots+\gamma^{n-1}R_{t+n}+\gamma^nv(S_{t+n}))}
\end{aligned}
$$

> Write down the sequence of $\lambda$-returns $v_t^\lambda$ corresponding to this episode for $\lambda=0.5$ and $b=2, l=-2$.

$$
\begin{aligned}
v_1^\lambda&=0.5\cdot (G_1^{(1)}+0.5\cdot G_1^{(2)}+0.25\cdot G_1^{3}) \\
&=0.5\cdot (-2+0.5\cdot 0+0.25\cdot 1)=-0.875 \\
v_2^\lambda&=0.5\cdot (G_2^{(1)}+0.5\cdot G_2^{(2)}) \\
&=0.5\cdot (0+0.5\cdot 1)=0.25 \\
v_3^\lambda&=0.5\cdot G_3^{(1)}=0.5 \\
\end{aligned}
$$

> Using the forward-view TD($\lambda$) algorithm and the previous linear function approximator, what are the sequence of updates to weight $b$? What is the total update to weight $b$? Using $\lambda=0.5, \gamma=1, \alpha=0.5$ and start with $b=2, l=-2$.

$$
\begin{aligned}
&\textit{According to the forward-view linear TD(}\lambda\textit{) parameter update rule,} \\\\
\Delta\mathbf{w}&=\alpha(G_t^\lambda-\hat{v}(S_t,\mathbf{w}))\mathbf{x}(S_t) \\\\
&\textit{we have the sequence of updates to weight }b\textit{ as below,} \\\\
\mathbf{w}_0&=\begin{pmatrix}b \\ l\end{pmatrix}=\begin{pmatrix}2 \\ -2\end{pmatrix} \\
\Delta\begin{pmatrix}b \\ l\end{pmatrix}_{t=1}&=\alpha(-0.875-2)\begin{pmatrix}1 \\ 0\end{pmatrix}=\begin{pmatrix}-1.4375 \\ 0\end{pmatrix},\mathbf{w}_1=\begin{pmatrix}0.5625 \\ -2\end{pmatrix} \\
\Delta\begin{pmatrix}b \\ l\end{pmatrix}_{t=2}&=\alpha(-0.46875+2)\begin{pmatrix}0 \\ 1\end{pmatrix}=\begin{pmatrix}0 \\ 0.765625\end{pmatrix},\mathbf{w}_2=\begin{pmatrix}0.5625 \\ -1.234375\end{pmatrix} \\
\Delta\begin{pmatrix}b \\ l\end{pmatrix}_{t=3}&=\alpha(0.5+0.671875)\begin{pmatrix}1 \\ 1\end{pmatrix}=\begin{pmatrix}0.5859375 \\ 0.5859375\end{pmatrix},\mathbf{w}_3=\begin{pmatrix}1.1484375 \\ -0.6484375\end{pmatrix} \\\\
&\textit{So the sequence of updates to weight b is }-1.4375,\ 0,\ 0.5859375\textit{, the total update is }-0.8515625.
\end{aligned}
$$

> Define the eligibility trace $e_t$ for TD($\lambda$) when using linear value function approximation.

$$
\begin{aligned}
e_0&=\mathbf{0} \\
e_t&=\gamma\lambda e_{t-1}+\mathbf{x}(S_t)
\end{aligned}
$$

> Write down the sequence of eligibility traces $e_t$ corresponding to bell, using $\lambda=0.5, \gamma=1$.

$$
\begin{aligned}
e_1(bell)&=1 \\
e_2(bell)&=0.5 \\
e_3(bell)&=1.25
\end{aligned}
$$

> Using the backward-view TD($\lambda$) algorithm and linear function approximator, what the sequence of updates and total update to weight $b$? (Use offline update and parameters $\lambda=0.5, \gamma=1, \alpha=0.5$ and start with $b=2, l=-2$.)

$$
\begin{aligned}
&\textit{The update rule for backward-view linear TD(}\lambda\textit{) is as below,} \\\\
\delta_t&=R_{t+1}+\gamma\hat{v}(S_{t+1},\mathbf{w})-\hat{v}(S_t,\mathbf{w}) \\
E_t&=\gamma\lambda E_{t-1}+\mathbf{x}(S_t) \\
\Delta\mathbf{w}&=\alpha\delta_tE_t \\\\
&\textit{since we will use offline update, the results in previous answer can be used here; we get the sequence of updates to weight b as below,} \\\\
\Delta b_1&=0.5\cdot(0-2-2)\cdot1=-2 \\
\Delta b_2&=0.5\cdot(0+0+2)\cdot0.5=0.5 \\
\Delta b_3&=0.5\cdot(1+0-0)\cdot1.25=0.625 \\\\
&\textit{so the total update to weight b is }\Delta b=-0.875.
\end{aligned}
$$
