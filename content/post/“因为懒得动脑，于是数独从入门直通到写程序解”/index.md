---
title: "“因为懒得动脑，于是数独从入门直通到写程序解”"
description: 
date: 2024-04-23T20:08:38+08:00
image: 
math: 
license: 
hidden: false
comments: false
isCJKLanguage: true
draft: false
---

事情是这样的，看了一篇[别人的数独入门的文章](https://rikumi.notion.site/Good-Sudoku-72033a35ca5a4449a981347e5a560bf4)。然后发现，技巧有点点多啊……啊，不想动脑，遂想直接写程序解决。**如果能用程序解决，为啥要动脑？**  

## 要想求解，那得先有题目

首先，如果要写出求解程序，那就必须先有一个生成数独题面的程序。（其实，这也就是针对题面全为空时的特殊求解程序）

总的算法是:

1. 往数独矩阵里一个一个地填数字。
2. 没有数字可以填的时候，就回溯上一个填数位置
3. 重复这个过程，直到整个矩阵被填满。

### 样例代码

```Python
from time import time
from random import Random
from collections import deque


def posToIdx(r, c):
    # Convert a position to 3*3 block index
    return (r // 3, c // 3)

class RestrictMat:
    def __init__(self):
        self.rowMat = {i: {x for x in range(1, 10)} for i in range(9)}
        self.colMat = {i: {x for x in range(1, 10)} for i in range(9)}
        # 3*3 sudoku block starting from (0, 0) at top-left
        self.blockMat = {(i, j): {x for x in range(1, 10)} for i in range(3) for j in range(3)}

    def drop(self, v, r, c):
        '''
        Drop a value from Restriction Matrix, which is the action of filling a number.
        param v: value
        param r: row number
        param c: col number
        '''
        self.rowMat[r].discard(v)
        self.colMat[c].discard(v)
        self.blockMat[posToIdx(r, c)].discard(v)

    def validNums(self, r, c):
        # return the current valid numbers to fill in
        return self.rowMat[r] & self.colMat[c] & self.blockMat[posToIdx(r, c)]

    def put(self, v, r, c):
        # Put back a number when a new number has been chosen for the pos
        self.rowMat[r].add(v)
        self.colMat[c].add(v)
        self.blockMat[posToIdx(r, c)].add(v)


def gen_map():
    m = [[0 for i in range(9)] for _ in range(9)]
    resMat = RestrictMat()
    rand = Random(time())
    stack = deque()
    stack.append(((0, 0), set()))

    while stack:
        (r, c), triedNums = stack.pop()
        candidates = list(resMat.validNums(r, c) - triedNums)
        if len(candidates) > 0:
            n = rand.choice(candidates)
            if m[r][c] != 0:
                resMat.put(m[r][c], r, c)
            m[r][c] = n
            resMat.drop(n, r, c)
            triedNums.add(n)
            stack.append(((r, c), triedNums))
            if r == 8 and c == 8:
                return m
            else:
                r += (c + 1) // 9
                c = (c + 1) % 9
                stack.append(((r, c), set()))
        else:
            if m[r][c] != 0:
                resMat.put(m[r][c], r, c)
                m[r][c] = 0
    return m

for r in gen_map():
    print(' '.join(map(str, r)))
```

### 解析

#### RestrictMatrix

首先，RestrictMatrix是一个根据数独规则，实时记录并更新矩阵中每一个位置可以填数字的哈希表约束矩阵。按照规则，可以保存三个与位置关系相对应的可放数字的哈希表：
- 横向不能有重复数字 >>> 按行保存可放的数字, **即其中的"rowMat"**

- 纵向不能有重复数字 >>> 按列保存可放的数字, **即其中的"colMat"**

- 所在的九宫格内不能有重复数字 >>> 按九宫格序号保存可放数字，**即其中的"blockMat"**

每一个位置实际可放的数字就是该位置的三个哈希表内容的交集（该位置三个哈希表共有的一系列数字）。每一次放入数字，便从三个哈希表的对应位置交集里取出对应数字。

这里使用了哈希表而没有像普通的约束编程一样用01矩阵，主要还是因为其调用方式简单明了。~~（我才不会说是因为我不怎么写编程题，所以不太会那种写法呢。）~~

##### \_\_init\_\_

初始化约束矩阵。最初，因为所有位置皆为空，所以都可以填任何数字。于是，全填1-9的数字。

```Python
def __init__(self):
    self.rowMat = {i: {x for x in range(1, 10)} for i in range(9)}
    self.colMat = {i: {x for x in range(1, 10)} for i in range(9)}
    # 3*3 sudoku block starting from (0, 0) at top-left
    self.blockMat = {(i, j): {x for x in range(1, 10)} for i in range(3) for j in range(3)}
```

##### drop

当在**r**行**c**列填入数字**v**时调用。更新当前整个盘面的可填数字。

```Python
def drop(self, v, r, c):
    '''
    Drop a value from Restriction Matrix, which is the action of filling a number.
    param v: value
    param r: row number
    param c: col number
    '''
    self.rowMat[r].discard(v)
    self.colMat[c].discard(v)
    self.blockMat[posToIdx(r, c)].discard(v)
```

这里使用discard来取出数字是因为要忽略掉

##### put

当**r**行**c**列取出数字**v**时调用。一般后一候选位置没有可填数字的时候，需要当前位置更改所填数字时发生。因为要换一个数字填，所以需要将此时已经填入的数字**v**从盘面上拿下来，再放回约束矩阵中。同时也需要更新候选数字。

```Python
def put(self, v, r, c):
    # Put back a number when a new number has been chosen for the pos
    self.rowMat[r].add(v)
    self.colMat[c].add(v)
    self.blockMat[posToIdx(r, c)].add(v)
```

##### validNums

将保存着**r**行**c**列三个约束条件的哈希表取交集，即可得到当前位置可填数字。

```Python
def validNums(self, r, c):
    # return the current valid numbers to fill in
    return self.rowMat[r] & self.colMat[c] & self.blockMat[posToIdx(r, c)]
```

#### 求解步骤

首先，需要初始化要用到的数据结构：数独盘面，一个约束矩阵还有一个用于记录当前已尝试了的位置和所填数的栈（每次尝试之后，都要把当前状态压入）。其中，如果位置填0,则表示该位置没有填入数字，为空位。

```Python
m = [[0 for i in range(9)] for _ in range(9)]
resMat = RestrictMat()
rand = Random(time())
stack = deque()
stack.append(((0, 0), set()))
```

开始求解的循环。以栈的容量为条件只是我的习惯罢了，后面会发现其实永远不为空，循环如果没有跳出条件会一直进行。

```Python
while stack:
```

每次开始时，都从栈中弹出当前要尝试填入的位置和当前位置已尝试过的数字的信息。第一次的时候肯定是(0,0)和 {} (空集)。之后随着不断尝试和填入下一个尝试位置会有所变化。

```Python
	(r, c), triedNums = stack.pop()
```

选出当前可供尝试的数字。这个候选数字是通过在当前位置可填的数字中，剔除已经尝试过了的数字得到的。

```Python
	candidates = list(resMat.validNums(r, c) - triedNums)
```

如果发现有可以填入的可尝试数字，则进行填入操作。

```Python
    if len(candidates) > 0:
        n = rand.choice(candidates)
        if m[r][c] != 0:
            resMat.put(m[r][c], r, c)
        m[r][c] = n
        resMat.drop(n, r, c)
        triedNums.add(n)
        stack.append(((r, c), triedNums))
```

其中，使用rand.choice来随机选中一个数字来填。

如果当前位置不为0，则说明之前已经填入过数字，需要取出放回约束矩阵。

```Python
        if m[r][c] != 0:
            resMat.put(m[r][c], r, c)
```

处理好之后，就可以进行填入数字的操作。

```Python
        m[r][c] = n
        resMat.drop(n, r, c)
```

还需要把这个数字记录到当前已经尝试过的表里，并压回栈中，以待后续填不下的时候弹出来重新选数重填。

```Python
        triedNums.add(n)
        stack.append(((r, c), triedNums))
```

填完之后，需要判断一下是否填完了。因为是从左往右从上往下的填入顺序，所以当当前位置是(8,8)的时候，即是最后位置。如果能填完数的话，就整个填完了，可以直接返回数独牌面跳出循环了。

```Python
        if r == 8 and c == 8:
            return m
```

若不是，则计算下一个位置，并压入栈中。等下次弹出时尝试。

```Python
        else:
            r += (c + 1) // 9
            c = (c + 1) % 9
            stack.append(((r, c), set()))
```

而假如当前位置已经没有可以尝试填入的数字时，则不继续往下尝试。而是取出当前已经填入的数（如果有的话）。

```Python
    # 对应之前有候选数的分支
    # if len(candidates) > 0:
	else:
        if m[r][c] != 0:
            resMat.put(m[r][c], r, c)
            m[r][c] = 0
```



最后，得到解的话，就可以打印数独盘面了。

```Python
for r in gen_map():
    print(' '.join(map(str, r)))
```

## 改写成一般求解程序

因为生成盘面的程序本身就是一种特殊情况下的求解程序。只需要将候选位置的更新算法和对应的记录方法修改即可改成一般求解程序。

### 代码

除了gen_map有所变化以外，其他都是一样的。所以不一一列出了。

```Python
def gen_map(m):
    resMat = RestrictMat()
    rand = Random(time())
    queue = deque()
    cnt, target = 0, 9 * 9

    for i in range(9):
        for j in range(9):
            if m[i][j] != 0:
                resMat.put(m[i][j], i, j)
                target -= 1
            else:
                queue.append(((i, j), set()))

    while queue:
        (r, c), triedNums = queue.pop()
        candidates = list(resMat.validNums(r, c) - triedNums)
        if len(candidates) > 0:
            n = rand.choice(candidates)
            if m[r][c] != 0:
                resMat.put(m[r][c], r, c)
            else:
                cnt += 1
            m[r][c] = n
            resMat.drop(n, r, c)
            triedNums.add(n)
            queue.append(((r, c), triedNums))
            if cnt == target:
                return m
            else:
                queue.append(queue.popleft())
        else:
            if m[r][c] != 0:
                resMat.put(m[r][c], r, c)
                m[r][c] = 0
                cnt -= 1
            queue.appendleft(((r, c), set()))
    return m
```

### 解析

首先，相比起生成盘面时已知的待求解数和待填位置，一般求解时需要首先分析出这些。只有填0的才是需要求解的位置，其他位置都不需要管，也不要动。

```Python
    cnt, target = 0, 9 * 9

    for i in range(9):
        for j in range(9):
            if m[i][j] != 0:
                resMat.put(m[i][j], i, j)
                target -= 1
            else:
                queue.append(((i, j), set()))
```

下一个尝试位置的更新以及求解完成的判定条件也有所改变。

当已填入位置数等于待求解数时，即完成。

位置更新方式改由从(queue)队尾取出一个待解位置放入队头，来让下次从队列里取位置的时候取出来。

```Python
    queue.append(((r, c), triedNums))
    if cnt == target:
        return m
    else:
        queue.append(queue.popleft())
```

而当前位置无候选数字的时候，便需要往队列里填回一个全新的待解位置。但不放入队头，因为需要回退待解位置的操作。

```Python
    else:
        if m[r][c] != 0:
            resMat.put(m[r][c], r, c)
            m[r][c] = 0
            cnt -= 1
        queue.appendleft(((r, c), set()))
```

## 后记 ~~（叠甲）~~

其实，我也知道我写的可能多少会有点可读性的问题，以及实现上欠考虑的地方。如果有啥意见或者改进建议，欢迎私聊笔者。~~(就算是这样，笔者起码也断断续续想了一周才写出来。我太菜了........)~~
