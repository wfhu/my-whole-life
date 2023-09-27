# successive over relaxation method（SOR）学习一例

背景：在阅读经典著作《Parallel and Distributed Computation: Numerical Methods》一书时，有提到relaxation method，特意学习一下

最初的信息来源：[https://zhuanlan.zhihu.com/p/25099638](https://zhuanlan.zhihu.com/p/25099638)

原书相关信息地址：[http://www.mit.edu/\~jnt/parallel.html](http://www.mit.edu/\~jnt/parallel.html)

参考youtube视频：[https://www.youtube.com/watch?v=D3S185K4WWc](https://www.youtube.com/watch?v=D3S185K4WWc)

参考python代码：[https://www.codesansar.com/numerical-methods/python-program-successive-over-relaxation.htm](https://www.codesansar.com/numerical-methods/python-program-successive-over-relaxation.htm)



看了上面文章和视频之后，自己手动演算了一次：

<figure><img src="../../../.gitbook/assets/SOR-pic.png" alt=""><figcaption><p>SOR-手动演算示例</p></figcaption></figure>

手动演算对真正理解非常有帮助，一定要自己亲自做一遍，我就在手动演算的过程中发现自己对什么时候用k+1以及k没有理解到位，演算结果出现了错误，马上重新理解+改正

用python代码实现：

```python
# successive over-relaxation (SOR)

#  2x + z = 6
#  2y + z = 3
#  y + 2z = 4.5

# Defining equations to be solved
# in diagonally dominant form
f1 = lambda x,y,z: (6-z)/2
f2 = lambda x,y,z: (3-z)/2
f3 = lambda x,y,z: (4.5-y)/2

# Initial setup
x0 = 0
y0 = 0
z0 = 0
count = 1

# Reading tolerable error
# e = float(input('Enter tolerable error: '))
e = 0.0000001

# Reading relaxation factor
# w = float(input("Enter relaxation factor: "))
w = 1.1

# Implementation of successive over-relaxation
print('\nCount\tx\ty\tz\n')

condition = True

while condition:
    x1 = (1-w) * x0 + w * f1(x0,y0,z0)
    y1 = (1-w) * y0 + w * f2(x1,y0,z0)
    z1 = (1-w) * z0 + w * f3(x1,y1,z0)
    print('%d\t%0.6f\t%0.6f\t%0.6f\n' %(count, x1,y1,z1))
    e1 = abs(x0-x1);
    e2 = abs(y0-y1);
    e3 = abs(z0-z1);
    
    count += 1
    x0 = x1
    y0 = y1
    z0 = z1
    
    condition = e1>e and e2>e and e3>e

print('\nSolution: x = %0.6f, y = %0.6f and z = %0.6f\n'% (x1,y1,z1))
```

运行结果如下：

```
Count	x	y	z

1	3.300000	1.650000	1.567500

2	2.107875	0.622875	1.975669

3	2.002595	0.501095	2.001831

4	1.998733	0.498883	2.000431

5	1.999890	0.499875	2.000026

6	1.999997	0.499998	1.999998

7	2.000001	0.500001	2.000000

8	2.000000	0.500000	2.000000

9	2.000000	0.500000	2.000000


Solution: x = 2.000000, y = 0.500000 and z = 2.000000
```



网上的代码，简单修改了一下：

```python
# successive over-relaxation (SOR)

#  4x - y + z = -1
#  -x + 4y - z = 7
#  x - y + 4z = -7

# Defining equations to be solved
# in diagonally dominant form
f1 = lambda x,y,z: (-1+y-z)/4
f2 = lambda x,y,z: (7+x+z)/4
f3 = lambda x,y,z: (-7-x+y)/4

# Initial setup
x0 = 0
y0 = 0
z0 = 0
count = 1

# Reading tolerable error
# e = float(input('Enter tolerable error: '))
e = 0.0000001

# Reading relaxation factor
# w = float(input("Enter relaxation factor: "))
w = 1.1

# Implementation of successive over-relaxation
print('\nCount\tx\ty\tz\n')

condition = True

while condition:
    x1 = (1-w) * x0 + w * f1(x0,y0,z0)
    y1 = (1-w) * y0 + w * f2(x1,y0,z0)
    z1 = (1-w) * z0 + w * f3(x1,y1,z0)
    print('%d\t%0.6f\t%0.6f\t%0.6f\n' %(count, x1,y1,z1))
    e1 = abs(x0-x1);
    e2 = abs(y0-y1);
    e3 = abs(z0-z1);
    
    count += 1
    x0 = x1
    y0 = y1
    z0 = z1
    
    condition = e1>e and e2>e and e3>e

print('\nSolution: x = %0.6f, y = %0.6f and z = %0.6f\n'% (x1,y1,z1))
```

运行输出结果如下：

```
Count	x	y	z

1	-0.275000	1.849375	-1.340797

2	0.629797	1.544538	-1.539367

3	0.510094	1.487496	-1.502278

4	0.496178	1.499573	-1.498839

5	0.499945	1.500347	-1.500006

6	0.500102	1.499992	-1.500030

7	0.499996	1.499991	-1.499998

8	0.499998	1.500001	-1.499999

9	0.500000	1.500000	-1.500000

10	0.500000	1.500000	-1.500000


Solution: x = 0.500000, y = 1.500000 and z = -1.500000
```

