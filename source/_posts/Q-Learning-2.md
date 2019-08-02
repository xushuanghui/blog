---
title: Q-Learning-2
date: 2019-04-01 16:46:27
tags:
---



## 算法更新

[![DQN 算法更新 (Tensorflow)](https://morvanzhou.github.io/static/results/reinforcement-learning/4-1-1.jpg)](https://morvanzhou.github.io/static/results/reinforcement-learning/4-1-1.jpg)

整个算法乍看起来很复杂, 不过我们拆分一下, 就变简单了. 也就是个 Q learning 主框架上加了些装饰.

这些装饰包括:

- 记忆库 (用于重复学习)
- 神经网络计算 Q 值
- 暂时冻结 `q_target` 参数 (切断相关性)
<!--more-->

##代码模式

首先我们先 import 两个模块, `maze_env` 是我们的环境模块,`maze_env` 模块我们可以不深入研究, 如果你对编辑环境感兴趣, 可以去看看如何使用 python 自带的简单 GUI 模块 `tkinter` 来编写虚拟环境.  `maze_env` 就是用 `tkinter` 编写的. 而 `RL_brain` 这个模块是 RL 的大脑部分, 我们下节会讲.

```
from maze_env import Maze
from RL_brain import QLearningTable
```

下面的代码, 我们可以根据上面的图片中的算法对应起来, 这就是整个 Qlearning 最重要的迭代更新部分啦.

```
def update():
    # 学习 100 回合
    for episode in range(100):
        # 初始化 state 的观测值
        observation = env.reset()

        while True:
            # 更新可视化环境
            env.render()

            # RL 大脑根据 state 的观测值挑选 action
            action = RL.choose_action(str(observation))

            # 探索者在环境中实施这个 action, 并得到环境返回的下一个 state 观测值, reward 和 done (是否是掉下地狱或者升上天堂)
            observation_, reward, done = env.step(action)

            # RL 从这个序列 (state, action, reward, state_) 中学习
            RL.learn(str(observation), action, reward, str(observation_))

            # 将下一个 state 的值传到下一次循环
            observation = observation_

            # 如果掉下地狱或者升上天堂, 这回合就结束了
            if done:
                break

    # 结束游戏并关闭窗口
    print('game over')
    env.destroy()

if __name__ == "__main__":
    # 定义环境 env 和 RL 方式
    env = Maze()
    RL = QLearningTable(actions=list(range(env.n_actions)))

    # 开始可视化环境 env
    env.after(100, update)
    env.mainloop()
```



## 代码主结构 

与上回不一样的地方是, 我们将要以一个 class 形式定义 Q learning, 并把这种 tabular q learning 方法叫做 `QLearningTable`.

```
class QLearningTable:
    # 初始化
    def __init__(self, actions, learning_rate=0.01, reward_decay=0.9, e_greedy=0.9):

    # 选行为
    def choose_action(self, observation):

    # 学习更新参数
    def learn(self, s, a, r, s_):

    # 检测 state 是否存在
    def check_state_exist(self, state):
```

## 预设值 

```
import numpy as np
import pandas as pd


class QLearningTable:
    def __init__(self, actions, learning_rate=0.01, reward_decay=0.9, e_greedy=0.9):
        self.actions = actions  # a list
        self.lr = learning_rate # 学习率
        self.gamma = reward_decay   # 奖励衰减
        self.epsilon = e_greedy     # 贪婪度
        self.q_table = pd.DataFrame(columns=self.actions, dtype=np.float64)   # 初始 q_table
```

## 决定行为 

这里是定义如何根据所在的 state, 或者是在这个 state 上的 观测值 (observation) 来决策.

```
    def choose_action(self, observation):
        self.check_state_exist(observation) # 检测本 state 是否在 q_table 中存在(见后面标题内容)

        # 选择 action
        if np.random.uniform() < self.epsilon:  # 选择 Q value 最高的 action
            state_action = self.q_table.loc[observation, :]

            # 同一个 state, 可能会有多个相同的 Q action value, 所以我们乱序一下
            action = np.random.choice(state_action[state_action == np.max(state_action)].index)

        else:   # 随机选择 action
            action = np.random.choice(self.actions)

        return action
```

## 学习 

我们根据是否是 `terminal` state (回合终止符) 来判断应该如何更行 `q_table`. 更新的方式是不是很熟悉呢:

`update = self.lr * (q_target - q_predict)`

这可以理解成神经网络中的更新方式, 学习率 * (真实值 - 预测值). 将判断误差传递回去, 有着和神经网络更新的异曲同工之处.

```
    def learn(self, s, a, r, s_):
        self.check_state_exist(s_)  # 检测 q_table 中是否存在 s_ (见后面标题内容)
        q_predict = self.q_table.loc[s, a]
        if s_ != 'terminal':
            q_target = r + self.gamma * self.q_table.loc[s_, :].max()  # 下个 state 不是 终止符
        else:
            q_target = r  # 下个 state 是终止符
        self.q_table.loc[s, a] += self.lr * (q_target - q_predict)  # 更新对应的 state-action 值
```

## 检测 state 是否存在 

这个功能就是检测 `q_table` 中有没有当前 state 的步骤了, 如果还没有当前 state, 那我我们就插入一组全 0 数据, 当做这个 state 的所有 action 初始 values.

```
    def check_state_exist(self, state):
        if state not in self.q_table.index:
            # append new state to q table
            self.q_table = self.q_table.append(
                pd.Series(
                    [0]*len(self.actions),
                    index=self.q_table.columns,
                    name=state,
                )
            )
```