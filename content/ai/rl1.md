---
title: "强化学习快餐教程(1) - gym环境搭建"
date: 2022-08-23T00:14:58+08:00
draft: false
---

欲练强化学习神功，首先得找一个可以操练的场地。
两大巨头OpenAI和Google DeepMind都不约而同的以游戏做为平台，比如OpenAI的长处是DOTA2，而DeepMind是AlphaGo下围棋。

下面我们就从OpenAI为我们提供的gym为入口，开始强化学习之旅。

## OpenAI gym平台安装

安装方法很简单，gym是python的一个包，通过pip安装即可。
例：
```
pip3 install gym --user
```

源代码的下载地址在：[https://github.com/openai/gym](https://github.com/openai/gym)

gym平台的目的就是提供一个供强化学习进行比较和交流的标准化平台。

## 第一个gym游戏：cart pole

cart pole是一个最简单的小游戏模型，它是一个一维的小车上竖起一根棍子，然后通过调整左右来保证棍子不倒。

我们先来一个随机输入的例子，大家先让这个小游戏跑起来：
```python
import gym
env = gym.make('CartPole-v0')
env.reset()
for _ in range(1000):
    env.render()
    env.step(env.action_space.sample()) # take a random action
env.close()
```

通过运行可以看到，别说棍子不倒了，绕着圈带着小车不知道飞到哪里去了。

gym主要为我们提供了两种元素：环境和操作。
我们首先通过gym.make来生成cartpole的运行环境，然后reset给小车和棍子一个初始化的值。
最后，通过env.step将操作传给环境去控制小车。

### 操作

cartpole的操作非常简单，只有两种命令，用0和1表示。0是向左推小车，1是向右推小车。小车是处在一个光滑轨道上的，根据牛顿第一定律，在无外力时处于静止或匀速直线运动的状态。

刚才我们调用env.action_space.sample()，就是在0和1之间随机生成两种状态之一做为输入。

除了刚才的随机策略外，我们也可以采取交替的策略：
```python
import gym
env = gym.make('CartPole-v0')
env.reset()
for _ in range(1000):
    i = 0
    env.render()
    env.step( (i+1) % 2) 
env.close()
```

### 获取环境信息

但是，有了操作之后是个开环的系统，我们需要通过从环境中读取信息来更好地决策。

其实，不管是reset还是step，环境都是会返回一系列值给我们的。

reset会返回一个状态信息给我们。而step会返回一个四元组，分别是状态信息，奖励信息，是否已经结束和附加信息。

我们进行一轮迭代，先判断下是否已经失败，如果已经倒了就结束游戏，然后统计一下我们活了几轮：
```python
import gym
env = gym.make('CartPole-v0')
status = env.reset()
for step in range(1000):
    i = 0
    env.render()
    status, reward, done, info = env.step( (i+1) % 2)
    if done: 
        print('dead in %d steps' % step)
        break
env.close()
```

下面我们再进一步，去读取一下状态信息。针对cartpole这个问题，状态信息还是一个4元组，分别是：
- 小车位置
- 小车速度
- 棍的倾斜角度
- 棍的角速度

在本游戏中，如果角度大于12度，或者小车位置超出了2.4，就意味着失败了，直接结束。

### 闭环控制

知道了反馈信息之后，我们就可以想办法进行闭环控制了。
比如我们只取位置参数，如果偏左了就向右推，反之亦然：

```python
def action(status): 
    pos, v, ang, va = status
    print(status)
    if pos <= 0: 
        return 1
    else: 
        return 0 
```

完整代码如下：
```python
import gym

def action(status): 
    pos, v, ang, va = status
    print(status)
    if pos <= 0: 
        return 1
    else: 
        return 0 

env = gym.make('CartPole-v0')
status = env.reset()
for step in range(1000):
    i = 0
    env.render()
    status, reward, done, info = env.step(action(status))
    if done: 
        print('dead in %d steps' % step)
        break
env.close()
```

下面是我运行一次的结果：
```
[-0.01635101  0.00400916 -0.02452805 -0.01815461]
[-0.01627082  0.19947413 -0.02489114 -0.31847439]
[-0.01228134  0.39494158 -0.03126063 -0.61890199]
[-0.00438251  0.59048591 -0.04363867 -0.9212643 ]
[ 0.00742721  0.78616953 -0.06206395 -1.22733599]
[ 0.0231506   0.59189892 -0.08661067 -0.95472622]
[ 0.03498858  0.3980421  -0.1057052  -0.69046268]
[ 0.04294942  0.2045334  -0.11951445 -0.43283924]
[ 0.04704009  0.0112884  -0.12817124 -0.18009313]
[ 0.04726586 -0.18178868 -0.1317731   0.0695676 ]
[ 0.04363008 -0.37479942 -0.13038175  0.31794448]
[ 0.0361341  -0.56784668 -0.12402286  0.56683387]
[ 0.02477716 -0.76103061 -0.11268618  0.81801468]
[ 0.00955655 -0.95444458 -0.09632589  1.07323592]
[-0.00953234 -1.14817057 -0.07486117  1.33420176]
[-0.03249575 -0.9521891  -0.04817713  1.01906429]
[-0.05153954 -0.7564593  -0.02779585  0.71165164]
[-0.06666872 -0.5609637  -0.01356281  0.41035059]
[-0.077888   -0.36565212 -0.0053558   0.11342282]
[-0.08520104 -0.17045384 -0.00308735 -0.180945  ]
[-0.08861011  0.02471216 -0.00670625 -0.47460027]
[-0.08811587  0.21992817 -0.01619825 -0.76938932]
[-0.08371731  0.41526928 -0.03158604 -1.06712463]
[-0.07541192  0.61079456 -0.05292853 -1.36955102]
[-0.06319603  0.80653727 -0.08031955 -1.67830763]
[-0.04706529  1.0024934  -0.1138857  -1.99488277]
[-0.02701542  1.19860803 -0.15378336 -2.32055915]
[-0.00304326  1.39475935 -0.20019454 -2.65634816]
dead in 27 steps
```

### 角策略

上一种策略我们是根据车的位置来进行控制。我们还可以考虑根据角度来进行控制：
```python
import gym

def action_a(status): 
    pos, v, ang, va = status
    print(status)
    if ang > 0: 
        return 1
    else: 
        return 0

env = gym.make('CartPole-v0')
status = env.reset()
for step in range(1000):
    i = 0
    env.render()
    status, reward, done, info = env.step(action_a(status))
    if done: 
        print('dead in %d steps' % step)
        break
env.close()
```

从一些次尝试来看，角策略比上一个位置策略要更优一些：
```
[ 0.00780229 -0.02463916 -0.01033269 -0.03445555]
[ 0.00730951 -0.21961142 -0.0110218   0.25494948]
[ 0.00291728 -0.41457429 -0.00592281  0.54413566]
[-0.00537421 -0.60961251  0.0049599   0.83494657]
[-0.01756646 -0.41455866  0.02165883  0.54382761]
[-0.02585763 -0.21974767  0.03253538  0.25804686]
[-0.03025258 -0.02510495  0.03769632 -0.02419899]
[-0.03075468  0.16945669  0.03721234 -0.30475403]
[-0.02736555  0.36402912  0.03111726 -0.58547272]
[-0.02008497  0.55870171  0.01940781 -0.86819324]
[-0.00891093  0.75355429  0.00204394 -1.15471154]
[ 0.00616015  0.94864953 -0.02105029 -1.44675287]
[ 0.02513314  0.75379272 -0.04998535 -1.16072073]
[ 0.040209    0.55935628 -0.07319976 -0.88411992]
[ 0.05139612  0.36530055 -0.09088216 -0.61531734]
[ 0.05870214  0.17155806 -0.10318851 -0.35258554]
[ 0.0621333  -0.02195676 -0.11024022 -0.09414094]
[ 0.06169416 -0.21534017 -0.11212304  0.16182831]
[ 0.05738736 -0.40869329 -0.10888647  0.41714169]
[ 0.04921349 -0.60211728 -0.10054364  0.67361   ]
[ 0.03717115 -0.7957087  -0.08707144  0.93302056]
[ 0.02125697 -0.98955482 -0.06841103  1.19712154]
[ 1.46587604e-03 -1.18372790e+00 -4.44685947e-02  1.46760271e+00]
[-0.02220868 -1.37827822 -0.01511654  1.74607025]
[-0.04977425 -1.57322512  0.01980486  2.03401309]
[-0.08123875 -1.37831278  0.06048513  1.74752417]
[-0.108805   -1.18392804  0.09543561  1.47425204]
[-0.13248357 -0.99009321  0.12492065  1.21283837]
[-0.15228543 -0.796785    0.14917742  0.96176679]
[-0.16822113 -0.60394842  0.16841275  0.71942016]
[-0.1803001  -0.41150733  0.18280116  0.48412211]
[-0.18853024 -0.21937201  0.1924836   0.25416579]
[-0.19291768 -0.02744473  0.19756691  0.02783298]
[-0.19346658  0.16437637  0.19812357 -0.19659389]
[-0.19017905  0.35619437  0.1941917  -0.42082427]
[-0.18305516  0.54811122  0.18577521 -0.64655444]
[-0.17209294  0.74022551  0.17284412 -0.87546311]
[-0.15728843  0.93262987  0.15533486 -1.10920578]
[-0.13863583  1.12540784  0.13315074 -1.34940608]
[-0.11612768  1.31862936  0.10616262 -1.59764218]
[-0.08975509  1.51234492  0.07420978 -1.85542638]
[-0.05950819  1.70657739  0.03710125 -2.12417555]
[-0.02537664  1.90131142 -0.00538226 -2.40517032]
[ 0.01264959  1.7062367  -0.05348567 -2.11414485]
[ 0.04677432  1.51168791 -0.09576856 -1.83845627]
[ 0.07700808  1.31774548 -0.13253769 -1.57698862]
[ 0.10336299  1.12442853 -0.16407746 -1.32840846]
[ 0.12585156  0.9317127  -0.19064563 -1.09123975]
dead in 47 steps
```

