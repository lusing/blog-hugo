---
title: "强化学习快餐教程(3) - 一条命令搞定atari游戏"
date: 2022-08-23T00:14:58+08:00
draft: false
---

通过上节的例子，我们试验出来，就算是像cartpole这样让一个杆子不倒这样的小模型，都不是特别容易搞定的。

那么像太空入侵者这么复杂的问题，建模都建不出来，算法该怎么写？

别急，我们从强化学习的基础来讲起，学习马尔可夫决策过程，了解贝尔曼方程、最优值函数、最优策略及其求解。然后学习动态规划法、蒙特卡洛法、时间差分法、值函数近似法、策略梯度法。再然后我们借用深度学习的武器来武装强化学习算法，我们会学习DQN算法族，讲解2013版的基于Replay Memory的DQN算法，还有2015年增加了Target网络的新DQN算法，还有Double DQN、优先级回放DQN和Dueling DQN，以及PG算法族的DPG，Actor-Critic，DDPG，以及A3C算法等等。

有的同学表示已经看晕了，除了堆了一堆名词什么也没看懂。别怕，我们程序员解决这样的问题统统是上代码说话，我们马上给大家一条命令就能解决太空入侵者以及其它更复杂的游戏。
有的同学表示还没看晕，嗯，这样的话后面我们增加一些推导公式和解读最新论文的环节 ：）

## baseline登场

下面，就是见证奇迹的时刻。对于太空入侵者这么复杂的问题，我们程序员用什么办法来解决它呢？答案是，调库！
库到哪里找呢？洪七公说过：凡毒蛇出没之处，七步内必有解救蛇毒之药。 其他毒物，无不如此，这是天地间万物生克的至理。OpenAI成天跟这些难题打交道，肯定有其解法。
没错，我们要用的库，就是openAI的baselines库。

调库之前，我们先下载源码：
```
git clone https://github.com/openai/baselines

```
然后安装一下库：
```
pip install -e .
```

见证奇迹的时刻到了，我们用一条命令，就可以解决这些复杂的atari游戏，以太空入侵者为例：
```
python -m baselines.run --alg=ppo2 --env=SpaceInvadersNoFrameskip-v4
```

baselines.run有两个参数：一个是算法，ppo2是OpenAI自家的最近策略优化算法Proximal Policy Optimization Algorithms；另一个是游戏环境，SpaceInvaders是太空入侵者游戏，后面的NoFrameskip-v4是控制参数，之前我们使用v0或v1中，为了模拟人的控制，不让控制太精准，所以都是有4帧左右的重复。而在训练的时候就去掉这个负优化，使训练的效率更高。

然后baselines就会和atari模拟器打交道去进行训练了，输入类似于下面这样：
```
Stepping environment...
Done.
--------------------------------------
| eplenmean               | 1.12e+03 |
| eprewmean               | 836      |
| fps                     | 721      |
| loss/approxkl           | 0.0043   |
| loss/clipfrac           | 0.144    |
| loss/policy_entropy     | 0.786    |
| loss/policy_loss        | -0.0137  |
| loss/value_loss         | 0.148    |
| misc/explained_variance | 0.946    |
| misc/nupdates           | 7.30e+03 |
| misc/serial_timesteps   | 9.35e+05 |
| misc/time_elapsed       | 1.08e+04 |
| misc/total_timesteps    | 7.48e+06 |
--------------------------------------
Stepping environment...
Done.
--------------------------------------
| eplenmean               | 1.12e+03 |
| eprewmean               | 836      |
| fps                     | 764      |
| loss/approxkl           | 0.00372  |
| loss/clipfrac           | 0.134    |
| loss/policy_entropy     | 0.713    |
| loss/policy_loss        | -0.0144  |
| loss/value_loss         | 0.118    |
| misc/explained_variance | 0.949    |
| misc/nupdates           | 7.31e+03 |
| misc/serial_timesteps   | 9.35e+05 |
| misc/time_elapsed       | 1.08e+04 |
| misc/total_timesteps    | 7.48e+06 |
--------------------------------------
Stepping environment...
Done.
--------------------------------------
| eplenmean               | 1.12e+03 |
| eprewmean               | 831      |
| fps                     | 749      |
| loss/approxkl           | 0.0039   |
| loss/clipfrac           | 0.13     |
| loss/policy_entropy     | 0.706    |
| loss/policy_loss        | -0.0143  |
| loss/value_loss         | 0.108    |
| misc/explained_variance | 0.941    |
| misc/nupdates           | 7.31e+03 |
| misc/serial_timesteps   | 9.35e+05 |
| misc/time_elapsed       | 1.08e+04 |
| misc/total_timesteps    | 7.48e+06 |
--------------------------------------
```
运行个一千万步左右，就可以看到分数的结果了。

如果我们不喜欢PPO算法，也可以换成其它的经典算法，比如DQN，把算法参数改成deepq就好。我们举个用DQN算法的例子，游戏我们也换成Breakout弹珠游戏吧。
```
python -m baselines.run --alg=deepq --env=BreakoutNoFrameskip-v4 --num_timesteps=1e6
```

大家注意到，我们还增加了第三个参数，执行多少步迭代的数目，这里我们选10万步。论文中一般选1000万步。

模型辛苦训练出来了，后面游戏中还要用呢，我们再增加一个参数保存起来：
```
python -m baselines.run --alg=ppo2 --env=SpaceInvadersNoFrameskip-v4 --num_timesteps=1e6 --save_path=./models/cartpole_1M_ppo2
```

## 使用深度Q学习解决cartpole问题

通过上面的样例，我们对于强化学习有了信心。下面我们学习用代码调用baselines的方法，我们首先以之前搞不定的cartpole那个立不住的杆子开始入手。

### 4步法训练深度强化模型

通过baselines调用强化学习库非常简单，只要4步就可以了：
第一步，创建cartpole游戏环境，这个我们已经非常熟悉了，使用gym.make来创建一个env
```python
env = gym.make("CartPole-v0")
```
第二步，确定训练结束的条件。这也跟库和算法无关，是领域知识。对于cartpole来说，能坚持199步不倒，也就是杆子的倾角不超过12度，且小车没有超出范围，就算是成功：
```python
def callback(lcl, _glb):
    # stop training if reward exceeds 199
    is_solved = lcl['t'] > 100 and sum(lcl['episode_rewards'][-101:-1]) / 100 >= 199
    return is_solved
```
第三步，选择一种强化学习算法并调用其训练方法，我们这里选用最经典的DQN算法：
```python
act = deepq.learn(
    env,
    network='mlp',
    lr=1e-3,
    total_timesteps=100000,
    buffer_size=50000,
    exploration_fraction=0.1,
    exploration_final_eps=0.02,
    print_freq=10,
    callback=callback
)
```
参数中，env是要用来训练的游戏环境，callback是我们刚才写的成功条件，total_timesteps是要练训多少步，我们选10万步就足够了。
第四步，将训练好的模型保存起来：
```python
act.save("cartpole_model.pkl")
```

经过这4步，我们就可以在一不懂算法原理，二不懂超参数是什么的情况下，成功解决cartpole问题，源码整合下如下：
```python
import gym

from baselines import deepq


def callback(lcl, _glb):
    # stop training if reward exceeds 199
    is_solved = lcl['t'] > 100 and sum(lcl['episode_rewards'][-101:-1]) / 100 >= 199
    return is_solved


env = gym.make("CartPole-v0")
act = deepq.learn(
    env,
    network='mlp',
    lr=1e-3,
    total_timesteps=100000,
    buffer_size=50000,
    exploration_fraction=0.1,
    exploration_final_eps=0.02,
    print_freq=10,
    callback=callback
)
print("Saving model to cartpole_model.pkl")
act.save("cartpole_model.pkl")
```

### 运行模型

模型训练好了，我们就拿来跑一下验证效果吧，我们把策略生成交给刚才的deepq.learn，其余的大家都已经很熟悉了，就是标准的运行游戏的过程：
```python
import gym

from baselines import deepq

env = gym.make("CartPole-v0")
act = deepq.learn(env, network='mlp', total_timesteps=0,load_path="cartpole_model.pkl")

obs, done = env.reset(), False
episode_rew = 0
while not done:
    env.render()
    obs, rew, done, _ = env.step(act(obs[None])[0])
    episode_rew += rew
    print("Episode reward", episode_rew)
```

从运行图上，我们就可以看到pole再也不倒了。从打印出来的reward结果上也证明了这一点：
```
Loaded model from cartpole_model.pkl
Episode reward 1.0
Episode reward 2.0
Episode reward 3.0
Episode reward 4.0
...
Episode reward 198.0
Episode reward 199.0
Episode reward 200.0
```
