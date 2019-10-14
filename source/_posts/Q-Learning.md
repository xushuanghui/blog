---
title: Q-Learning-1
date: 2019-03-29 20:06:02
tags: 强化学习
---



###Q-Learning 整体算法

[![Q Leaning](https://morvanzhou.github.io/static/results/ML-intro/q4.png)](https://morvanzhou.github.io/static/results/ML-intro/q4.png)

 Q learning 的算法, 每次更新我们都用到了 Q 现实和 Q 估计, 而且 Q learning 的迷人之处就是 在 Q(s1, a2) 现实 中, 也包含了一个 Q(s2) 的最大估计值, 将对下一步的衰减的最大估计和当前所得到的奖励当成这一步的现实, 很奇妙吧. 最后我们来说说这套算法中一些参数的意义. Epsilon greedy 是用在决策上的一种策略, 比如 epsilon = 0.9 时, 就说明有90% 的情况我会按照 Q 表的最优值选择行为, 10% 的时间使用随机选行为. alpha是学习率, 来决定这次的误差有多少是要被学习的, alpha是一个小于1 的数. gamma 是对未来 reward 的衰减值. 我们可以这样想象.

<!--more-->



## Q-Learning 更新

[![Q Leaning](https://morvanzhou.github.io/static/results/ML-intro/q3.png)](https://morvanzhou.github.io/static/results/ML-intro/q3.png)





## Q-Learning Gamma

![gamma](https://morvanzhou.github.io/static/results/ML-intro/q5.png)

## 例子:

```
-o---T
# T 就是宝藏的位置, o 是探索者的位置
```

Q-learning 是一种记录行为值 (Q value) 的方法, 每种在一定状态的行为都会有一个值 `Q(s, a)`, 就是说 行为 `a` 在 `s` 状态的值是 `Q(s, a)`. `s` 在上面的探索者游戏中, 就是 `o` 所在的地点了. 而每一个地点探索者都能做出两个行为 `left/right`, 这就是探索者的所有可行的 `a` 啦.

如果在某个地点 `s1`, 探索者计算了他能有的两个行为, `a1/a2=left/right`, 计算结果是 `Q(s1, a1) > Q(s1, a2)`, 那么探索者就会选择 `left` 这个行为. 这就是 Q learning 的行为选择简单规则.

## 预设值

这一次需要的模块和参数设置:

```
import numpy as np
import pandas as pd
import time

N_STATES = 6   # 1维世界的宽度
ACTIONS = ['left', 'right']     # 探索者的可用动作
EPSILON = 0.9   # 贪婪度 greedy
ALPHA = 0.1     # 学习率
GAMMA = 0.9    # 奖励递减值
MAX_EPISODES = 13   # 最大回合数
FRESH_TIME = 0.3    # 移动间隔时间
```

## Q 表

对于 tabular Q learning, 我们必须将所有的 Q values (行为值) 放在 `q_table` 中, 更新 `q_table` 也是在更新他的行为准则. `q_table` 的 index 是所有对应的 `state` (探索者位置), columns 是对应的 `action` (探索者行为).

```
def build_q_table(n_states, actions):
    table = pd.DataFrame(
        np.zeros((n_states, len(actions))),     # q_table 全 0 初始
        columns=actions,    # columns 对应的是行为名称
    )
    return table

# q_table:
"""
   left  right
0   0.0    0.0
1   0.0    0.0
2   0.0    0.0
3   0.0    0.0
4   0.0    0.0
5   0.0    0.0
"""
```

## 定义动作

接着定义探索者是如何挑选行为的. 这是我们引入 `epsilon greedy` 的概念. 因为在初始阶段, 随机的探索环境, 往往比固定的行为模式要好, 所以这也是累积经验的阶段, 我们希望探索者不会那么贪婪(greedy). 所以 `EPSILON` 就是用来控制贪婪程度的值. `EPSILON` 可以随着探索时间不断提升(越来越贪婪), 不过在这个例子中, 我们就固定成 `EPSILON = 0.9`, 90% 的时间是选择最优策略, 10% 的时间来探索.

```
# 在某个 state 地点, 选择行为
def choose_action(state, q_table):
    state_actions = q_table.iloc[state, :]  # 选出这个 state 的所有 action 值
    if (np.random.uniform() > EPSILON) or (state_actions.all() == 0):  # 非贪婪 or 或者这个 state 还没有探索过
        action_name = np.random.choice(ACTIONS)
    else:
        action_name = state_actions.argmax()    # 贪婪模式
    return action_name
```

## 环境反馈 S_, R

做出行为后, 环境也要给我们的行为一个反馈, 反馈出下个 state (S_) 和 在上个 state (S) 做出 action (A) 所得到的 reward (R). 这里定义的规则就是, 只有当 `o` 移动到了 `T`, 探索者才会得到唯一的一个奖励, 奖励值 R=1, 其他情况都没有奖励.

```
def get_env_feedback(S, A):
    # This is how agent will interact with the environment
    if A == 'right':    # move right
        if S == N_STATES - 2:   # terminate
            S_ = 'terminal'
            R = 1
        else:
            S_ = S + 1
            R = 0
    else:   # move left
        R = 0
        if S == 0:
            S_ = S  # reach the wall
        else:
            S_ = S - 1
    return S_, R
```

## 环境更新

接下来就是环境的更新了, 不用细看.

```
def update_env(S, episode, step_counter):
    # This is how environment be updated
    env_list = ['-']*(N_STATES-1) + ['T']   # '---------T' our environment
    if S == 'terminal':
        interaction = 'Episode %s: total_steps = %s' % (episode+1, step_counter)
        print('\r{}'.format(interaction), end='')
        time.sleep(2)
        print('\r                                ', end='')
    else:
        env_list[S] = 'o'
        interaction = ''.join(env_list)
        print('\r{}'.format(interaction), end='')
        time.sleep(FRESH_TIME)
```

## 强化学习主循环

最重要的地方就在这里. 你定义的 RL 方法都在这里体现. 在之后的教程中, 我们会更加详细得讲解 RL 中的各种方法, 下面的内容, 大家大概看看就行, 这节内容不用仔细研究.

[![小例子](https://morvanzhou.github.io/static/results/reinforcement-learning/2-1-1.png)](https://morvanzhou.github.io/static/results/reinforcement-learning/2-1-1.png)

```
def rl():
    q_table = build_q_table(N_STATES, ACTIONS)  # 初始 q table
    for episode in range(MAX_EPISODES):     # 回合
        step_counter = 0
        S = 0   # 回合初始位置
        is_terminated = False   # 是否回合结束
        update_env(S, episode, step_counter)    # 环境更新
        while not is_terminated:

            A = choose_action(S, q_table)   # 选行为
            S_, R = get_env_feedback(S, A)  # 实施行为并得到环境的反馈
            q_predict = q_table.loc[S, A]    # 估算的(状态-行为)值
            if S_ != 'terminal':
                q_target = R + GAMMA * q_table.iloc[S_, :].max()   #  实际的(状态-行为)值 (回合没结束)
            else:
                q_target = R     #  实际的(状态-行为)值 (回合结束)
                is_terminated = True    # terminate this episode

            q_table.loc[S, A] += ALPHA * (q_target - q_predict)  #  q_table 更新
            S = S_  # 探索者移动到下一个 state

            update_env(S, episode, step_counter+1)  # 环境更新

            step_counter += 1
    return q_table
```

写好所有的评估和更新准则后, 我们就能开始训练了, 把探索者丢到环境中, 让它自己去玩吧.

```
if __name__ == "__main__":
    q_table = rl()
    print('\r\nQ-table:\n')
    print(q_table)
```