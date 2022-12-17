---
layout: post
title:  "Nahamcon CTF 22: Rektcursion"
date:   2022-12-17 08:02:59 +0100
categories: jekyll update
---
In rektcursion we get the following function
```python
def f(i):
    if i < 5:
        return i+1
    
    return 1905846624*f(i-5) - 133141548*f(i-4) + 3715204*f(i-3) - 51759*f(i-2) + 360*f(i-1)
```

And we are asked to compute
	f(13371337)
This recursion takes a lot of time and memory and is kinda infeasible.
The hint in the problem is to diagonalize a matrix.
Here my mathematics education comes in handy!

A recursive series can be written as a matrix. In the example above:
```python
A = [[360 -51759 3715204 -133141548 1905846624],
     [1    0     0         0                 0],
     [0    1     0         0                 0],
     [0    0     1         0                 0],
     [0    0     0         1                 0]]
```
Multiplying this matrix with a vector containing [f(i-1) f(i-2) f(i-3) f(i-4) f(i-5)] it all makes sense
Calculating A*[5 4 3 2 1] gives f(5)
so calculating A^(13371333)*[5 4 3 2 1] should give us f(13371337)

Matrix exponentials are hard except for some simple cases such as diagonal matrices. We use the decomposition
	A = P * D * P^(-1)

where D is a diagonal matrix.
Then:
	A^2 = P * D^2 * P^(-1)

I used sage to calculate it. Unfortunately we can't work mod 10^31337 so the numbers get kinda big. Also I didn't have sage installed so hilarity ensued when I copy pasted a 25MB int from a sagemathcell server into notepad.
```
A = matrix([[360, -51759, 3715204, -133141548, 1905846624],
     [1,    0,     0,         0,                 0],
     [0,    1,     0,         0,                 0],
     [0,    0,     1,         0,                 0],
     [0,    0,     0,         1,                 0]])
D = A.jordan_form()
D=D**13371333
P = A.eigenmatrix_right()[1]
Pi = P.inverse()
A = P*D*Pi
v = vector([5,4,3,2,1])
(A*v)[0]
```
