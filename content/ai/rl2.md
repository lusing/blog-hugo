---
title: "强化学习快餐教程(2) - atari游戏"
date: 2022-08-23T00:14:58+08:00
draft: false
---

不知道看了上节的内容，大家有没有找到让杆不倒的好算法。
现在我们晋阶一下，向世界上第一种大规模的游戏机atari前进。

## 太空入侵者

可以通过
```
pip install atari_py
```
来安装atari游戏。

下面我们以SpaceInvaders-v0为例看下Atari游戏的环境的特点。

### 图形版

在太空入侵者中，支持的输入有6种，一个是什么也不做，一个是开火，另4个是控制方向：
-    0: NOOP
-    1: FIRE
-    2: UP
-    3: RIGHT
-    4: LEFT
-    5: DOWN

我们从环境中获取的信息是什么呢？很不幸，是一个(210, 160, 3)的图片，显示出来是这样的：
![初始环境](https://upload-images.jianshu.io/upload_images/1638145-831a719592711f10.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们写代码把这个环境搭起来。策略嘛，我就原地不动一直开火。

```python
import gym
from skimage import io
env = gym.make('SpaceInvaders-v0')
status = env.reset()

for step in range(1000):
    env.render()
    thisstep = 1
    status, reward, done, info = env.step(thisstep)
    jpgname = './pic-%d.jpg' % step
    io.imsave(jpgname,status)
    print(reward)
    if done: 
        print('dead in %d steps' % step)
        break
env.close()
```

大家可以通过保存下来的pic-x.jpg来直观观察游戏的情况，比如我在第138步时，打中了一个5分的入侵者。
![pic-138.jpg](https://upload-images.jianshu.io/upload_images/1638145-fe1f1e7de0f71cd4.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

太空入侵者这个游戏的策略比起cartpole，需要分析图像，这个是不足。但是它也是有好处的，就是reward参数现在会把分数返回给我们。总算是分数处理上不需要搞图像分析了。

下面我来写个算法吧，以1/4的概率左右移动，另外3/4开火：
```python
import gym
env = gym.make('SpaceInvaders-v0')
status = env.reset()

def policy(step): 
    state_pool = [3,4,3,3,4,4,3,3,3,4,4,4,3,3,3,3,4,4,4,4]
    if step % 4 == 0: 
        pos = step / 4
        result = state_pool[int(pos % (len(state_pool)))] 
        return result
    else: 
        return 1

for step in range(10000):
    env.render()
    thisstep = policy(step)
    print(thisstep)
    status, reward, done, info = env.step(thisstep)
    #print(reward)
    if done: 
        print('dead in %d steps' % step)
        break
env.close()
```

### 内存版

如果图像分析做起来不方便的话，gym还为我们提供了RAM版的。就是将游戏机中的128个字节的内存信息提供给我们。
下面是env.reset的128个字节的例子：
```
[  0   7   0  68 241 162  34 183  68  13 124 255 255  50 255 255   0  36
  63  63  63  63  63  63  82   0  23  43  35 117 180   0  36  63  63  63
  63  63  63 110   0  23   1  60 126 126 126 126 255 255 255 195  60 126
 126 126 126 255 255 255 195  60 126 126 126 126 255 255 255 195   0   0
  48   3 129   0   0   0   0   0   0 246 246  63  63 246 246  63  63   0
  21  24   0  52  82 196 246  20   7   0 226   0   0   0   0   0  21  63
   0 128 171   0 255   0 189   0   0   0   0   0  99 255   0   0 235 254
 192 242]
```

输入的部分跟图像版是一样的，我们代码修改如下：
```python
import gym
env = gym.make('SpaceInvaders-ram-v0')
status = env.reset()
print(status)

def policy(step): 
    state_pool = [3,4,3,3,4,4,3,3,3,4,4,4,3,3,3,3,4,4,4,4]
    if step % 4 == 0: 
        pos = step / 4
        result = state_pool[int(pos % (len(state_pool)))] 
        return result
    else: 
        return 1

for step in range(10000):
    env.render()
    thisstep = policy(step)
    #print(thisstep)
    status, reward, done, info = env.step(thisstep)
    #print(reward)
    if done: 
        print('dead in %d steps' % step)
        break
env.close()
```

## breakout

下面我们再来一个弹球游戏。

![breakout-0.jpg](https://upload-images.jianshu.io/upload_images/1638145-980042676aff5e99.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

弹球游戏的输入是4个值。

图片版的：
```python
import gym
from skimage import io
env = gym.make('Breakout-v0')
status = env.reset()
#print(status)
print(env.action_space)

def policy(step):
    if step % 2 == 0:  
        return 2
    else: 
        return 3

for step in range(100):
    env.render()
    thisstep = policy(step)
    #print(thisstep)
    status, reward, done, info = env.step(thisstep)
    jpgname = './pic-%d.jpg' % step
    io.imsave(jpgname,status)

    #print(reward)
    if done: 
        print('dead in %d steps' % step)
        break
env.close()
```

RAM版的例子：
```python
import gym
env = gym.make('Breakout-ram-v0')
status = env.reset()
print(status)
print(env.action_space)

def policy(step): 
    return step % 4

for step in range(100):
    env.render()
    thisstep = policy(step)
    #print(thisstep)
    status, reward, done, info = env.step(thisstep)
    #print(reward)
    if done: 
        print('dead in %d steps' % step)
        break
env.close()
```

RAM的初始值：
```
[ 63  63  63  63  63  63 255 255 255 255 255 255 255 255 255 255 255 255
 255 255 255 255 255 255 255 255 255 255 255 255 192 192 192 192 192 192
 255 255 255 255 255 255 255 255 255 255 255 255 255 240   0   0 255   0
   0 240   0   5   0   0   6   0  70 182 134 198  22  38  54  70  88   6
 146   0   8   0   0   0   0   0   0 241   0 242   0 242  25 241   5 242
   0   0 255   0 228   0   0   0   0   0   0   0   0   0   0   0   0   0
   8   0 255 255 255 255 255 255 255   0   0   5   0   0 186 214 117 246
 219 242]
```
