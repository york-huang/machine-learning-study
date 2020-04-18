# AlphaZero Gomoku学习笔记

[AlphaZero Gomoku](https://github.com/junxiaosong/AlphaZero_Gomoku)根据AlphaZero的相关[文章](https://science.sciencemag.org/content/362/6419/1140)，实现了五子棋的AlphaZero下棋程序，研究它的代码旨在更好地理解RL应用在对弈游戏程序中的若干关键理论：MCTS、Minimax、Actor-Critic、self-play以及基于深度网络的策略/价值函数参数化技术。

## 从AlphaGo到AlphaZero

简单的比较：

* AlphaGo：MCTS、基于self-play与大量现成棋谱（即先验知识、专家知识）的数据训练、基于深度网络的策略/价值函数，仅针对围棋。

* AlphaGo Zero：比起AlphaGo，仅以self-play来产生训练数据，不使用先验知识。

* AlphaZero：不使用先验知识，一个更泛化的框架，除围棋外还适用于国际象棋、日本将棋。

## 技术要点

* 棋盘局面数据结构：$4*width*height$ tensor

* MCTS树结构

* MCTS中特殊的基于先验概率、访问次数以及Q值的greedy法选择子结点

* MCTS中递归的值符号交替反转的叶结点价值回溯传播

* 分别输出局面价值与策略的策略价值网络

* 策略价值网络的特殊损失函数

* rollout与self-play生成训练数据

* 真实对局中的下子决策机制：rollout多次根据结点访问次数来选择最优的下一子着法

## 主要源码文件简析

* `human_play.py`：提供人机对弈接口，默认使用已训练好的策略网络模型文件`best_policy_8_8_5.model`及使用AlphaZero机制的MCTS算法`mcts_alphaZero.MCTSPlayer`来下子，每下一子进行400次模拟对局（playout）。

* `game.py`：定义基础数据结构Board类以提供对局面状态（`Board.current_state`）、下子后状态更新（`Board.do_move`）、胜负评判（`Board.game_end`）和双方下子记录的封装，定义Game类以封装人机对弈（`Game.start_play`）和自对弈（`Game.start_self_play`）的流程逻辑。

* `train.py`：定义训练管线TrainPipeline类，包含自对弈及训练数据收集（`TrainPipeline.collect_selfplay_data`）、数据扩充（`TrainPipeline.get_equi_data`）、策略更新（`TrainPipeline.policy_update`）和策略评估（`TrainPipeline.policy_evaluate`）等核心训练功能。

* `mcts_alphaZero.py`：定义MCTS树结点结构TreeNode类、封装各核心MCTS操作（playout、价值回溯更新、计算各下子概率）的MCTS类、封装对弈接口的MCTSPlayer类。

* `mcts_pure.py`：实现纯MCTS的下子算法，主要用于策略价值网络训练过程的策略评估（即比较使用AlphaZero的MCTS与简单的MCTS的棋力以评判当前策略价值网络的训练效果）。

* `policy_value_net_*.py`：策略价值网络的结构定义以及迭代训练接口，不同文件对应不同的机器学习后端，包括Keras、Tensoflow、PyTorch与Theano。

## 棋盘局面数据结构

* width列height行的棋盘，定义左下角位置坐标为(0,0)，右上角坐标为(width-1,height-1)。称下子为move，move为一整数，则下子位置(n,m)对应的move值是：$move=n*width+m$。

* 两棋手以序号0和1标识，棋手0或1都可以先手，在`Game.start_play`方法的start\_player指定，在人机对弈模式下，默认AI先手。出现和棋时则以-1为胜者序号。

* Board对象以states字典维护move=>player\_index的映射（即记录哪位棋手下了哪个move），以availables数组存储未用的move值（即可着子的位置），以current\_player记录准备着下一子的棋手序号，以last\_move记录上一子的move值，每下一子，则在`Board.do_move`中更新上面提及的全部变量：添加新映射到states，从availables中移除已用的move，更新current\_player与last\_move。

* 在`Board.current_state`方法中计算并返回一$4*width*height$ tensor，四个矩阵的定义是：

  * tensor[0]：在current\_player有着子的位置置1，其余位置为0。

  * tensor[1]：在另一棋手有着子的位置置1，其余位置为0。

  * tensor[2]：在last\_move对应的位置置1，其余位置为0。

  * tensor[3]：如果current\_player是先手的棋手，则全部置1，否则置0。

## MCTS搜索树结构

树结点成员变量解释：

* \_parent：指向父结点。

* \_children：move=>node字典，维护move到各子结点的映射。

* \_n\_visits：本结点在历次playout中的累计被访问次数。

* \_Q：结点的Q价值。

* \_u：计算结点价值（不同于Q价值，而是用于在greedy法中选择结点时所用的价值）时的中间变量。

* \_P：选择本结点的先验概率（即从父节点选择执行本结点对应的move而到达本结点的先验概率）。

结点价值的计算，`TreeNode.get_value`：

```Python
self._u = (c_puct * self._P * np.sqrt(self._parent._n_visits) / (1 + self._n_visits))
return self._Q + self._u
```

其中c\_puct为传入参数，代码中可见默认值为5，根据上式，在结点访问次数\_n\_visits尚少的情况下（即迭代更新次数少），参数c\_puct、先验概率\_P与父结点访问次数\_parent.\_n\_visits会显著影响结点价值的计算，而随着self.\_n\_visits的增长，结点价值会趋近于本身的Q价值。

在从上向下进行树搜索过程中的子结点选择（选择子结点实际上是选择子结点对应的move）算法，`TreeNode.select`，则是简单的选取`TreeNode.get_value`返回值最大的子结点，结合上述价值计算的分析，可以认为这是特殊的贪心算法：在结点迭代更新次数少的时候，各结点的先验概率对树搜索起很大的作用，而随着迭代增加，树搜索算法回归到单纯的Q价值贪心算法。

结点迭代更新是通过叶结点价值回溯传播实现的，代码可见使用了递归法，见：

```Python
def update(self, leaf_value):
    """Update node values from leaf evaluation.
    leaf_value: the value of subtree evaluation from the current player's
        perspective.
    """
    # Count visit.
    self._n_visits += 1
    # Update Q, a running average of values for all visits.
    self._Q += 1.0*(leaf_value - self._Q) / self._n_visits

def update_recursive(self, leaf_value):
    """Like a call to update(), but applied recursively for all ancestors.
    """
    # If it is not root, this node's parent should be updated first.
    if self._parent:
        self._parent.update_recursive(-leaf_value)
    self.update(leaf_value)
```

代码中的叶结点价值由rollout或策略价值网络获得（见下文解说），取值为1（当前棋手获胜）、-1或0（和棋）。由代码可见，沿着模拟对局（playout）路径的所有结点的\_n\_visits与\_Q在回溯中会被更新，而通过在每次递归中都反转叶结点价值的符号来实现当前棋手与对弈棋手Q价值（平均值）的增减，即对应Minimax算法中对弈双方的价值在搜索树的各层中交替地往正负两个方向的累积，有所不同的是，上述方法巧妙地使得从上向下搜索结点时，统一使用最大价值贪心算法就能等价地实现传统Minimax搜索中的最大值最小值逐层交替进行的操作。

## 纯MCTS下子算法

`mcts_pure.py`实现了纯MCTS的下子算法，基本原理是模拟对局（playout）多次（默认10000次），取累计访问次数最多的子结点对应的move为着下一子的位置，解析如下：

* 下子决策`MCTS.get_move`

输入棋盘局面，执行\_n\_playout次playout，其中每次执行时传入playout的是棋盘局面的一个copy，即确保了原棋盘局面不会被playout所改变，这样每次playout所使用的棋盘局面都是一样的，即对同一个输入棋盘局面重复\_n\_playout次模拟对局。

模拟对局结束，MCTS搜索树各结点保存了对应的访问计数与Q值等信息，此时取根结点中有最大访问数的子结点所对应的move为下子位置，返回该move，并重置MCTS搜索树（即删除所有树结点，放弃之前从模拟对局中获得的信息）。

* 模拟对局流程`MCTS._playout`

  * __搜索阶段__：从根结点出发，依照`TreeNode.select`规则不断向下搜索子结点直到到达叶结点，与此同时，每选择一子结点，则以其对应的move去更新棋盘局面（`Board.do_move`）。

  * 若棋盘局面到达终结状态，则以局面的胜负值，从当前结点（即__搜索阶段__结束时到达的叶结点）开始递归回溯更新对局路径上所有结点的信息（\_n\_visits与Q值），本次模拟对局完结。

  * 若棋盘局面并未终结，则把当前局面下所有可能的下一子位置添加为当前结点（见上一点描述）的子结点，并令各子结点的先验概率服从均匀分布，即$\frac{1}{number\ of\ Board.availables)}$。

    * __rollout阶段__：从当前结点开始，完全随机地下子（从可着子的位置选择move），每下一子则更新棋盘局面，直到棋局终结，这一过程称为rollout。

    * 以rollout获得的胜负结果，从当前结点开始递归回溯更新结点信息直至根结点，注意，这里的当前结点同上文的解释，即是rollout开始前的结点，也就是__搜索阶段__结束时到达的结点。

总结纯MCTS下子策略：输入当前棋盘局面，通过多次模拟对局为其从根开始建立全新的一棵MCTS搜索树，然后把根结点下有最多访问次数的子结点对应的move作为下一子的位置返回。在模拟对局中，树搜索阶段使用特殊的贪心算法（如前文所述），添加子结点时则添加所有可能的下一move对应的子结点并赋予相同的先验概率，在rollout阶段则进行完全随机的下子，最后把对局胜负值递归回溯更新路径上各结点的信息。

## AlphaZero MCTS下子算法

`mcts_alphaZero.py`实现了基于AlphaZero策略价值网络的MCTS，但其主要数据结构与纯MCTS几乎无异，不同点体现在模拟对局与选子策略：

* 搜索树结点数据结构相同。

* 递归回溯更新结点信息法相同。

* 结点价值计算方式相同。

* 模拟对局（playout）实现方式不同，在AlphaZero算法下，playout在经历了__搜索阶段__后，若当前局面并未终结，则从策略价值网络中根据当前棋盘局面获得下一步所有可能的move及其对应的先验概率，并以之扩充当前结点的子结点，与此同时，策略价值网络还根据局面计算一个叶结点价值（称为局面评分可能更恰当，其值介乎于-1与1之间，意义与纯MCTS的__rollout阶段__获得的终局胜负得分相当），并以此来递归回溯更新相关结点的信息。

* 通过多次模拟对局（playout），从根结点的子结点集中选取下一子move的思路与纯MCTS一致，但具体实现方法很不同：

  * 多次playout后（默认2000次），从根结点的子结点集中根据访问计数算出各move对应的概率，公式是：$move\_probabilities = softmax(\frac{1}{temperature}\ *\ \log(array\_of\_node\_visits\_count))$，其中temperature常数默认取值为0.001。

  * 若当前处于人机对弈模式，则直接依据move\_probabilities选择下一子move返回，并清空MCTS搜索树。

  * 若当前处于自对弈模式（用于训练策略价值网络），则先对move\_probabilities添加狄利克雷噪声，再根据带噪声的概率选择下一子move来返回，与此同时，保留MCTS搜索树，但把根结点往下推进一层，以选中的move对应的子结点为新根结点。添加狄利克雷噪声的代码为：

    * ```Python
      p=0.75*probs + 0.25*np.random.dirichlet(0.3*np.ones(len(probs)))
      ```

可见，AlphaZero下的MCTS，模拟对局无rollout过程，MCTS结点信息的更新依赖于策略价值网络直接计算得到的move概率与局面评分，这是它与纯MCTS最大的不同之处。

## 策略价值网络结构

策略价值网络，Policy Value Net，即以深度神经网络的方式实现策略与价值函数的参数化，输入棋盘局面状态，分别输出动作概率（即move概率）与局面评分，其结构包含若干卷积层作为网络结构的公共部分，在此之上分别构建出动作网络（Action Networks）与局面评价网络（Evaluation Networks），解析如下（以`policy_value_net_tensorflow.py`为依据）：

* 输入tensor维度：$[batch,width,height,4]$，即batch size个棋盘局面状态，其中4出现在最后是因为Tensorflow多个API的data_format参数默认为“channels_last”；输入tensor的解释与生成见前文提及的`Board.current_state`。

* 公共层

  * 卷积层1：32个filter，卷积核尺寸3\*3，激活函数ReLU，输出维度为$[batch,width,height,32]$。

  * 卷积层2：64个filter，卷积核尺寸3\*3，激活函数ReLU，输出维度为$[batch,width,height,64]$。

  * 卷积层3：128个filter，卷积核尺寸3\*3，激活函数ReLU，输出维度为$[batch,width,height,128]$。

* 动作网络

  * 动作卷积层：以卷积层3为输入，包含4个filter，卷积核尺寸1\*1，激活函数ReLU，输出维度为$[batch,width,height,4]$。

  * 动作卷积扁平层：扁平化变维操作，输出维度为$[batch,4*width*height]$。

  * 动作全连接层：激活函数log-softmax，输出维度为$[batch,width*height]$，至此，可见输出即为各move的概率。

* 局面评价网络

  * 评价卷积层：以卷积层3为输入，包含2个filter，卷积核尺寸1\*1，激活函数ReLU，输出维度为$[batch,width,height,2]$。

  * 评价卷积扁平层：扁平化变维操作，输出维度为$[batch,2*width*height]$。

  * 评价全连接层1：激活函数ReLU，输出维度为$[batch,64]$。

  * 评价全连接层2：激活函数tanh，输出维度为$[batch,1]$，至此，可见输出即为对局面的评分，取值范围$[-1,1]$。

策略价值网络（包括动作网络与局面评价网络两个子网络）的损失函数由三部分相加而成：

* 价值损失，即局面评价网络输出（局面评分）与实际终局得分（胜、负、和）的平均平方误差（MSE），代码是：

  * ```Python
    self.value_loss = tf.losses.mean_squared_error(self.labels, self.evaluation_fc2)
    ```

* 策略损失，动作网络输出（move概率）与自对弈MCTS搜索树输出的move概率的交叉熵（cross entropy），公式和代码（注意动作网络的激活函数是log-softmax，即输出的move概率已经过log操作）分别为：

  * $CEH(p,q)=\mathbb{E}_p[-\log q]=-\sum\limits_{x\in\mathcal{X}}{p(x)\log q(x)}$

  * ```Python
    self.policy_loss = tf.negative(tf.reduce_mean(
                tf.reduce_sum(tf.multiply(self.mcts_probs, self.action_fc), 1)))
    ```

* L2正则项，用于避免参数过大，代码是：

  * ```Python
    l2_penalty_beta = 1e-4
    vars = tf.trainable_variables()
    l2_penalty = l2_penalty_beta * tf.add_n([tf.nn.l2_loss(v) for v in vars if 'bias' not in v.name.lower()])
    ```

与此同时，作者还定义了策略熵以用于评价训练效果，对应代码如下：

* ```Python
  self.entropy = tf.negative(tf.reduce_mean(tf.reduce_sum(tf.exp(self.action_fc) * self.action_fc, 1)))
  ```

最后，最小化损失函数的算法使用了常见的Adam梯度下降法。

## 训练流程

先解析AlphaZero MCTS的自对弈流程，这是获取训练数据的手段：

* 每一局自对弈，在每下一子之前，都会生成下面的数据：

  * 当前棋盘局面状态，即前面介绍过的$[4, width, height]$ tensor。

  * 由AlphaZero MCTS进行400次playout，生成下一子的所有move概率，同时选出最优move。
  
  * 标识当前棋手的棋手序号。
  
* 暂存上述数据，以AlphaZero MCTS选出的最优move走子，更新棋盘局面。

* 重复上述数据生成与走子过程，直到棋盘局面终结，得出胜负值。

* 把终局胜负值与该局所有走子数据绑定（注意，依据暂存起来的每一子的当前棋手标识，让胜者对应的胜负值为胜，败者对应的胜负值为负），获得形如$[array\ of\ states, array\ of\ move\ probabilities, array\ of\ winning]$的episode数据，其中：

  * state维度是：$[4, width, height]$。

  * move probability维度是：$[width*height,]$。
  
  * winning维度是：$[1,]$。

可以看到自对弈数据以一局为一个episode，每个episode包含对局里面每下一子时的棋盘局面、下子概率以及终局胜负，当然终局胜负是在对局结束后回填的。同时，利用棋盘局面的对称性，代码作者对每一个episode数据都进行了多种旋转和翻转，使得扩充后数据为原数据量的八倍，见`TrainPipeline.get_equi_data`。

有了上面的准备工作，可以梳理策略价值网络的训练流程如下（训练管线代码见`train.py`）：

* 网络训练全过程需经历1500局自对弈。

* 每局自对弈，先以AlphaZero MCTS生成前述的episode数据，并扩充之，存入训练数据池。

* 当训练数据池中episode数超过512后，每完成一局自对弈，则从训练数据池中随机抽取512个episode数据作为一个训练batch。

* 每个batch，进行最多5个epoch的训练以更新策略价值网络的参数，并以下面的形式进行KL散度测量：

  * 开始第一个epoch训练之前，以现有网络计算关于这个batch的move概率与局面评分，记为“旧move概率”与“旧局面评分”。
  
  * 每进行一个epoch的训练及参数更新后，重新计算一次move概率与局面评分，并计算与旧move概率之间的KL散度，代码：

    * ```Python
      kl = np.mean(np.sum(old_probs * (np.log(old_probs + 1e-10) - np.log(new_probs + 1e-10)), axis=1))
      ```
  
  * 若KL散度大于KL散度目标值（默认值0.02）的四倍，则跳出改batch的训练，即使未完成全部5个epoch的训练。

* 训练完一个batch后，根据最后一次算出的KL散度与K散度目标值的比较关系，更新学习率乘数，代码：

  * ```Python
    if kl > self.kl_targ * 2 and self.lr_multiplier > 0.1:
        self.lr_multiplier /= 1.5
    elif kl < self.kl_targ / 2 and self.lr_multiplier < 10:
        self.lr_multiplier *= 1.5
    ```

* 每完成50局自对弈，则做一次模型胜率评估，评估算法是：

  * 以基于当前策略价值网络的AlphaZero MCTS与纯MCTS对弈10局，双方轮流先手，其中纯MCTS每走一子进行1000次playout；以AlphaZero MCTS胜一局记1分，和局记0.5分，计算胜率。

  * 当出现历史最高胜率时，则把当前策略价值网络的模型参数存盘。

  * 当胜率达到100%时，则增加纯MCTS下子的playout次数，并重置最佳胜率，代码：

    * ```Python
      if (self.best_win_ratio == 1.0 and self.pure_mcts_playout_num < 5000):
          self.pure_mcts_playout_num += 1000
          self.best_win_ratio = 0.0
      ```

## Tensorflow笔记

研读AlphaZero Gomoku的策略价值网络部分代码时，主要参考的是Tensorflow实现，下面记录一些当时产生过疑惑的概念：

### logit究竟是什么意思？

在概率论里面logit的含义与odds相关，odds的定义是：$odds=\frac{probability\ of\ event}{probability\ of\ no\ event}$，而$logit=\log(odds)$。但在Tensorflow的语境中，logit并不能按此理解，现在直接摘录[StackOverflow上的回答](https://stackoverflow.com/questions/34240703/what-is-logits-softmax-and-softmax-cross-entropy-with-logits?noredirect=1&lq=1%5D)来解释：

> Logits simply means that the function operates on the unscaled output of earlier layers and that the relative scale to understand the units is linear. It means, in particular, the sum of the inputs may not equal 1, that the values are not probabilities (you might have an input of 5).
> `tf.nn.softmax` produces just the result of applying the softmax function to an input tensor. The softmax "squishes" the inputs so that sum(output) = 1: it's a way of normalizing. The shape of output of a softmax is the same as the input: it just normalizes the values. The outputs of softmax can be interpreted as probabilities.

```Python
a = tf.constant(np.array([[.1, .3, .5, .9]]))
print s.run(tf.nn.softmax(a))
[[ 0.16838508  0.205666    0.25120102  0.37474789]]
```

> In contrast, `tf.nn.softmax_cross_entropy_with_logits` computes the cross entropy of the result after applying the softmax function (but it does it all together in a more mathematically careful way). It's similar to the result of:

```Python
sm = tf.nn.softmax(x)
ce = cross_entropy(sm)
```

> The cross entropy is a summary metric: it sums across the elements. The output of `tf.nn.softmax_cross_entropy_with_logits` on a shape [2,5] tensor is of shape [2,1] (the first dimension is treated as the batch).
> If you want to do optimization to minimize the cross entropy AND you're softmaxing after your last layer, you should use `tf.nn.softmax_cross_entropy_with_logits` instead of doing it yourself, because it covers numerically unstable corner cases in the mathematically right way. Otherwise, you'll end up hacking it by adding little epsilons here and there.
> If you have single-class labels, where an object can only belong to one class, you might now consider using `tf.nn.sparse_softmax_cross_entropy_with_logits` so that you don't have to convert your labels to a dense one-hot array. This function was added after release 0.6.0.

### 卷积层若干参数的意义，特别是kernel_size参数与实际进行卷积时候的kernel维度问题？

Assuming original shape of the input is [batch, 4, height, width], why we need the following transposition?

```Python
self.input_state = tf.transpose(self.input_states, [0, 2, 3, 1])
```

Answer: the input transposition corresponds to this layer definitions:

```Python
self.conv1 = tf.layers.conv2d(inputs=self.input_state,
    filters=32, kernel_size=[3, 3],
    padding="same", data_format="channels_last",
    activation=tf.nn.relu)
```

* data\_format="channels_last" means the shape of the input is (batch, height, width, channels), so the transposition is necessary.

* padding='same' and default stribes=(1,1) make the 2d output shape unchanged, i.e., (height, width).

* true kernel shape is (kernel height, kernel width, input channels), in the above example, it's (3, 3, 4).

* number of filters is the number of kernels, also it is the exact number of output channels, i.e., 32 kernels which is of the shape (3, 3, 4), number of output channels is 32.

* so the above conv1 pipeline is like [number of batches, height, width, input channels] ==> [number of batches, height, width, output channels], i.e., [batch, height, width, 4] ==> [batch, height, width, 32].

* the math behind each output "pixel" is computed as $ReLU( dot( flattened\ kernel\ weights[3*3*4], flattened\ input\ data\ window[3*3*4] ) + bias)$, the total number of parameters of the above conv1 example is: $(3 * 3 * 4 + 1) * 32 = 1184$.
