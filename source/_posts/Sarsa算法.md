---
title: Sarsa算法
date: 2019-04-01 17:27:08
tags: 强化学习
---

## Sarsa 更新行为准则 

[![Sarsa](https://morvanzhou.github.io/static/results/ML-intro/s3.png)](https://morvanzhou.github.io/static/results/ML-intro/s3.png)

同样, 我们会经历正在写作业的状态 s1, 然后再挑选一个带来最大潜在奖励的动作 a2, 这样我们就到达了 继续写作业状态 s2, 而在这一步, 如果你用的是 Q learning, 你会观看一下在 s2 上选取哪一个动作会带来最大的奖励, 但是在真正要做决定时, 却不一定会选取到那个带来最大奖励的动作, Q-learning 在这一步只是估计了一下接下来的动作值. 而 Sarsa 是实践派, 他说到做到, 在 s2 这一步估算的动作也是接下来要做的动作. 所以 Q(s1, a2) 现实的计算值, 我们也会稍稍改动, 去掉maxQ, 取而代之的是在 s2 上我们实实在在选取的 a2 的 Q 值. 最后像 Q learning 一样, 求出现实和估计的差距 并更新 Q 表里的 Q(s1, a2).

## 对比 Sarsa 和 Q-learning 算法 

[![Sarsa](https://morvanzhou.github.io/static/results/ML-intro/s4.png)](https://morvanzhou.github.io/static/results/ML-intro/s4.png)

从算法来看, 这就是他们两最大的不同之处了. 因为 Sarsa 是说到做到型, 所以我们也叫他 on-policy, 在线学习, 学着自己在做的事情. 而 Q learning 是说到但并不一定做到, 所以它也叫作 Off-policy, 离线学习. 而因为有了 maxQ, Q-learning 也是一个特别勇敢的算法.

[![Sarsa](https://morvanzhou.github.io/static/results/ML-intro/s5.png)](https://morvanzhou.github.io/static/results/ML-intro/s5.png)

为什么说他勇敢呢, 因为 Q learning 机器人 永远都会选择最近的一条通往成功的道路, 不管这条路会有多危险. 而 Sarsa 则是相当保守, 他会选择离危险远远的, 拿到宝藏是次要的, 保住自己的小命才是王道. 这就是使用 Sarsa 方法的不同之处.



## 算法 

[![Sarsa 算法更新](https://morvanzhou.github.io/static/results/reinforcement-learning/3-1-1.png)](https://morvanzhou.github.io/static/results/reinforcement-learning/3-1-1.png)

整个算法还是一直不断更新 Q table 里的值, 然后再根据新的值来判断要在某个 state 采取怎样的 action. 不过于 Qlearning 不同之处:

- 他在当前 `state` 已经想好了 `state` 对应的 `action`, 而且想好了 下一个 `state_` 和下一个 `action_` (Qlearning 还没有想好下一个 `action_`)
- 更新 `Q(s,a)` 的时候基于的是下一个 `Q(s_, a_)` (Qlearning 是基于 `maxQ(s_)`)

这种不同之处使得 Sarsa 相对于 Qlearning, 更加的胆小. 因为 Qlearning 永远都是想着 `maxQ` 最大化, 因为这个 `maxQ` 而变得贪婪, 不考虑其他非 `maxQ` 的结果. 我们可以理解成 Qlearning 是一种贪婪, 大胆, 勇敢的算法, 对于错误, 死亡并不在乎. 而 Sarsa 是一种保守的算法, 他在乎每一步决策, 对于错误和死亡比较铭感. 这一点我们会在可视化的部分看出他们的不同. 两种算法都有他们的好处, 比如在实际中, 你比较在乎机器的损害, 用一种保守的算法, 在训练时就能减少损坏的次数.

## 算法的代码形式 

首先我们先 import 两个模块, `maze_env` 是我们的环境模块, 已经编写好了, `maze_env` 模块，我们可以不深入研究, 如果你对编辑环境感兴趣, 可以去看看如何使用 python 自带的简单 GUI 模块 `tkinter` 来编写虚拟环境.`maze_env` 就是用 `tkinter` 编写的. 而 `RL_brain` 这个模块是 RL 的大脑部分, 我们下节会讲.

```
from maze_env import Maze
from RL_brain import SarsaTable
```

下面的代码, 我们可以根据上面的图片中的算法对应起来, 这就是整个 Sarsa 最重要的迭代更新部分啦.

```
def update():
    for episode in range(100):
        # 初始化环境
        observation = env.reset()

        # Sarsa 根据 state 观测选择行为
        action = RL.choose_action(str(observation))

        while True:
            # 刷新环境
            env.render()

            # 在环境中采取行为, 获得下一个 state_ (obervation_), reward, 和是否终止
            observation_, reward, done = env.step(action)

            # 根据下一个 state (obervation_) 选取下一个 action_
            action_ = RL.choose_action(str(observation_))

            # 从 (s, a, r, s, a) 中学习, 更新 Q_tabel 的参数 ==> Sarsa
            RL.learn(str(observation), action, reward, str(observation_), action_)

            # 将下一个当成下一步的 state (observation) and action
            observation = observation_
            action = action_

            # 终止时跳出循环
            if done:
                break

    # 大循环完毕
    print('game over')
    env.destroy()

if __name__ == "__main__":
    env = Maze()
    RL = SarsaTable(actions=list(range(env.n_actions)))

    env.after(100, update)
    env.mainloop()
```



## 代码主结构 

和之前定义 Qlearning 中的 `QLearningTable` 一样, 因为使用 tabular 方式的 `Sarsa` 和 `Qlearning` 的相似度极高,

```
class SarsaTable:
    # 初始化 (与之前一样)
    def __init__(self, actions, learning_rate=0.01, reward_decay=0.9, e_greedy=0.9):

    # 选行为 (与之前一样)
    def choose_action(self, observation):

    # 学习更新参数 (有改变)
    def learn(self, s, a, r, s_):

    # 检测 state 是否存在 (与之前一样)
    def check_state_exist(self, state):
```

我们甚至可以定义一个 主class `RL`, 然后将 `QLearningTable` 和 `SarsaTable` 作为 主class `RL` 的衍生, 这个主 `RL` 可以这样定义. 所以我们将之前的 `__init__`, `check_state_exist`, `choose_action`, `learn` 全部都放在这个主结构中, 之后根据不同的算法更改对应的内容就好了. 所以还没弄懂这些功能的朋友们, 请回到之前的教程再看一遍.

```
import numpy as np
import pandas as pd


class RL(object):
    def __init__(self, action_space, learning_rate=0.01, reward_decay=0.9, e_greedy=0.9):
        ... # 和 QLearningTable 中的代码一样

    def check_state_exist(self, state):
        ... # 和 QLearningTable 中的代码一样

    def choose_action(self, observation):
        ... # 和 QLearningTable 中的代码一样

    def learn(self, *args):
        pass # 每种的都有点不同, 所以用 pass
```

如果是这样定义父类的 `RL` class, 通过继承关系, 那之子类 `QLearningTable` class 就能简化成这样:

```
class QLearningTable(RL):   # 继承了父类 RL
    def __init__(self, actions, learning_rate=0.01, reward_decay=0.9, e_greedy=0.9):
        super(QLearningTable, self).__init__(actions, learning_rate, reward_decay, e_greedy)    # 表示继承关系

    def learn(self, s, a, r, s_):   # learn 的方法在每种类型中有不一样, 需重新定义
        self.check_state_exist(s_)
        q_predict = self.q_table.loc[s, a]
        if s_ != 'terminal':
            q_target = r + self.gamma * self.q_table.loc[s_, :].max()
        else:
            q_target = r
        self.q_table.loc[s, a] += self.lr * (q_target - q_predict)
```

## 学习 

有了父类的 `RL`, 我们这次的编写就很简单, 只需要编写 `SarsaTable` 中 `learn` 这个功能就完成了. 因为其他功能都和父类是一样的. 这就是我们所有的 `SarsaTable` 于父类 `RL` 不同之处的代码. 是不是很简单.

```
class SarsaTable(RL):   # 继承 RL class

    def __init__(self, actions, learning_rate=0.01, reward_decay=0.9, e_greedy=0.9):
        super(SarsaTable, self).__init__(actions, learning_rate, reward_decay, e_greedy)    # 表示继承关系

    def learn(self, s, a, r, s_, a_):
        self.check_state_exist(s_)
        q_predict = self.q_table.loc[s, a]
        if s_ != 'terminal':
            q_target = r + self.gamma * self.q_table.loc[s_, a_]  # q_target 基于选好的 a_ 而不是 Q(s_) 的最大值
        else:
            q_target = r  # 如果 s_ 是终止符
        self.q_table.loc[s, a] += self.lr * (q_target - q_predict)  # 更新 q_table
```