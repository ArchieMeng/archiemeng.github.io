---
title: "Too Lazy to Use Brainpower, So Sudoku: From Beginner to Writing Code for Solutions"
description: 
date: 2024-05-18T15:15:30+08:00
image: 
math: 
license: 
hidden: false
comments: false
isCJKLanguage: true
draft: false
categories: ["Tech"]
tags: [
    "Sudoku",
    "Python",
    "Algorithm",
    "Constraint Programming",
]
---

So, here's the thing. I read [someone else's introductory article on Sudoku](https://rikumi.notion.site/Good-Sudoku-72033a35ca5a4449a981347e5a560bf4). I realized there were quite a few techniques… Ah, I didn't want to think too much, so I decided to write a program to solve it instead. Why use brain power when a program can do at ease?

## To solve it, you need a puzzle first

First of all, to write a solving program, you need a program to generate a Sudoku puzzle. (Actually, this is a special case of solving a puzzle where the grid is completely empty.)

The general algorithm is:

1. Fill numbers into the Sudoku grid one by one.
2. If there is no number that can be filled, backtrack to the previous position.
3. Repeat this process until the entire grid is filled.

### Sample code

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

### Analysis

#### RestrictMatrix

First, RestrictMatrix is a hash table constraint matrix that records and updates the valid numbers that can be filled in each position of the grid in real-time according to Sudoku rules. According to the rules, it can save three hash tables of valid numbers corresponding to the position relationships:

- No repeated numbers in rows >>> Save valid numbers by row, **which is "rowMat"**
- No repeated numbers in columns >>> Save valid numbers by column, **which is "colMat"**
- No repeated numbers within the 3x3 subgrid >>> Save valid numbers by subgrid index, **which is "blockMat"**

The actual numbers that can be placed in each position are the intersection of these three hash tables at that position (a series of numbers that are common to the three hash tables at that position). Each time a number is placed, it is taken out of the intersection of the corresponding positions of the three hash tables.

Using a hash table instead of a 0-1 matrix, a data structure commonly  used in ordinary constraint programming, is mainly because its usage is  simple and straightforward. ~~(I'm not saying it's because I rarely write programming problems, so I'm not very good at that method.)~~

##### \_\_init\_\_

Initialize the constraint matrix. Initially, since all positions are  empty, any number can be filled. So, fill all with numbers 1-9.

```Python
def __init__(self):
    self.rowMat = {i: {x for x in range(1, 10)} for i in range(9)}
    self.colMat = {i: {x for x in range(1, 10)} for i in range(9)}
    # 3*3 sudoku block starting from (0, 0) at top-left
    self.blockMat = {(i, j): {x for x in range(1, 10)} for i in range(3) for j in range(3)}
```

##### drop

Called when a number **v** is placed in row **r**, column **c**. Update the valid numbers for the entire grid.

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

Here, **discard** is used to remove the number because we need to ignore the absence of the number.

##### put

Called when the number **v** is removed from row **r**, column **c**. Generally happens when there are no numbers to fill in the next  candidate position and the current position needs to change the filled  number. Since a different number needs to be filled, the number **v** that has been filled needs to be removed from the grid and put back  into the constraint matrix. Also, the candidate numbers need to be  updated.

```Python
def put(self, v, r, c):
    # Put back a number when a new number has been chosen for the pos
    self.rowMat[r].add(v)
    self.colMat[c].add(v)
    self.blockMat[posToIdx(r, c)].add(v)
```

##### validNums

Take the intersections of the three constraint hash tables for row **r** and column **c** to get the numbers that can be placed in the current position.

```Python
def validNums(self, r, c):
    # return the current valid numbers to fill in
    return self.rowMat[r] & self.colMat[c] & self.blockMat[posToIdx(r, c)]
```

#### Solving steps

First, initialize the data structures to be used: the Sudoku grid, a  constraint matrix, and a stack to record the current position and  numbers tried (each time after trying, push the current state into the  stack). If the position is filled with 0, it indicates that the position is empty and no number has been filled.

```Python
m = [[0 for i in range(9)] for _ in range(9)]
resMat = RestrictMat()
rand = Random(time())
stack = deque()
stack.append(((0, 0), set()))
```

Start the solving loop. Using the stack size as a condition is just my habit; you'll find out later that it actually never empties, and the loop will continue if there's no exit condition.

```Python
while stack:
```

At the beginning of each iteration, pop the current position and the  numbers tried from the stack. At the first time, it will definitely be (0,  0) and {} (empty set). After continuous attempts and filling the next  position, it will change.

```Python
	(r, c), triedNums = stack.pop()
```

Select the numbers that can be tried for the current position. These candidate numbers are obtained by removing the numbers already tried  from the possible numbers for the current position.

```Python
	candidates = list(resMat.validNums(r, c) - triedNums)
```

If there are numbers that can be tried, proceed with filling.

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

Here, **rand.choice** is used to randomly select a number to fill.

If the current position is not 0, it means a number has been filled before, and it needs to be removed and put back into the constraint matrix.

```Python
        if m[r][c] != 0:
            resMat.put(m[r][c], r, c)
```

After the checks, you can proceed with filling the number.

```Python
        m[r][c] = n
        resMat.drop(n, r, c)
```

Also, record this number in the set of numbers tried. Push it back into  the stack so it can be popped out and tried again when unable to proceed

```Python
        triedNums.add(n)
        stack.append(((r, c), triedNums))
```

After filling, check if it is complete. Since the filling order is from left to right and top to bottom, when the current position is (8, 8), it enters the last position. If the number can be filled, the entire grid is  filled, and the Sudoku grid can be returned, exiting the loop.

```Python
        if r == 8 and c == 8:
            return m
```

If not, calculate the next position and push it into the stack to be tried next time.

```Python
        else:
            r += (c + 1) // 9
            c = (c + 1) % 9
            stack.append(((r, c), set()))
```

If there are no numbers that can be tried at the current position, do  not continue but remove the number (if any) that has been filled.

```Python
    # 对应之前有候选数的分支
    # if len(candidates) > 0:
	else:
        if m[r][c] != 0:
            resMat.put(m[r][c], r, c)
            m[r][c] = 0
```

Finally, if a solution is obtained, print the Sudoku grid.

```Python
for r in gen_map():
    print(' '.join(map(str, r)))
```

## Rewriting into a general solving program

Since the program to generate the grid is essentially a special case of  the solving program, it only needs to modify the candidate position  update algorithm and the corresponding recording method to turn it into a general solving program.

### Code

Only the **gen_map** function changes; the rest are exactly the same, so they are not listed one by one.

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

### Analysis

First, compared to generating a grid where the numbers to be solved and  the positions to be filled are known, general solving requires analyzing these first. Only positions filled with 0 need to be solved; others do not need to be touched or moved.

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

The update of the next position and the condition for determining if the solution is complete have also changed.

When the number of positions filled equals the number to be solved, it is complete.

The position update method is changed by taking a position to be solved from the end of the queue and placing it at the front, so that it can be taken out next time from the queue.

```Python
    queue.append(((r, c), triedNums))
    if cnt == target:
        return m
    else:
        queue.append(queue.popleft())
```

When there are no candidate numbers at the current position, a new  position to be solved needs to be filled back into the queue but not at  the front, because backtracking is needed.

```Python
    else:
        if m[r][c] != 0:
            resMat.put(m[r][c], r, c)
            m[r][c] = 0
            cnt -= 1
        queue.appendleft(((r, c), set()))
```

## Postscript

Actually, I know that my writing might have some readability issues and  implementation considerations. If you have any comments or suggestions  for improvement, feel free to message me privately. ~~(Even so, it took me about a week to come up with this intermittently. I'm so bad...)~~
