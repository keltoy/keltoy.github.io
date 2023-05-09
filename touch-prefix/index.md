# Torch 预热


```python
import torch

```


```python
# 行向量
x = torch.arange(12)
x
```




    tensor([ 0,  1,  2,  3,  4,  5,  6,  7,  8,  9, 10, 11])




```python
# 张量形状
x.shape

```




    torch.Size([12])




```python
# 元素总数
x.numel()

```




    12




```python
# 改变顺序
X = x.reshape(3, 4)
X
```




    tensor([[ 0,  1,  2,  3],
            [ 4,  5,  6,  7],
            [ 8,  9, 10, 11]])




```python
# 全零
torch.zeros((2, 3, 4))

```




    tensor([[[0., 0., 0., 0.],
             [0., 0., 0., 0.],
             [0., 0., 0., 0.]],
    
            [[0., 0., 0., 0.],
             [0., 0., 0., 0.],
             [0., 0., 0., 0.]]])




```python
# 全1
torch.ones((2, 3, 4))
```




    tensor([[[1., 1., 1., 1.],
             [1., 1., 1., 1.],
             [1., 1., 1., 1.]],
    
            [[1., 1., 1., 1.],
             [1., 1., 1., 1.],
             [1., 1., 1., 1.]]])




```python
# 随机张量
torch.randn(3, 4)
```




    tensor([[ 1.2938,  0.0115, -1.1682, -0.8837],
            [ 0.5345, -0.3015, -1.2261,  0.0236],
            [ 1.4058, -1.2990,  0.5318, -0.9359]])




```python
# 创建张量
torch.tensor([[2, 1, 4, 3], [1, 2, 3, 4], [4, 3, 2, 1]])

```




    tensor([[2, 1, 4, 3],
            [1, 2, 3, 4],
            [4, 3, 2, 1]])




```python
# 四则运算
x = torch.tensor([1.0, 2, 4, 8])
y = torch.tensor([2, 2, 2, 2])
x + y, x - y, x * y, x / y, x ** y  # **运算符是求幂运算
```




    (tensor([ 3.,  4.,  6., 10.]),
     tensor([-1.,  0.,  2.,  6.]),
     tensor([ 2.,  4.,  8., 16.]),
     tensor([0.5000, 1.0000, 2.0000, 4.0000]),
     tensor([ 1.,  4., 16., 64.]))




```python
torch.exp(x)

```




    tensor([2.7183e+00, 7.3891e+00, 5.4598e+01, 2.9810e+03])




```python
# 连接两个张量
X = torch.arange(12, dtype=torch.float32).reshape((3,4))
Y = torch.tensor([[2.0, 1, 4, 3], [1, 2, 3, 4], [4, 3, 2, 1]])
torch.cat((X, Y), dim=0), torch.cat((X, Y), dim=1)
```




    (tensor([[ 0.,  1.,  2.,  3.],
             [ 4.,  5.,  6.,  7.],
             [ 8.,  9., 10., 11.],
             [ 2.,  1.,  4.,  3.],
             [ 1.,  2.,  3.,  4.],
             [ 4.,  3.,  2.,  1.]]),
     tensor([[ 0.,  1.,  2.,  3.,  2.,  1.,  4.,  3.],
             [ 4.,  5.,  6.,  7.,  1.,  2.,  3.,  4.],
             [ 8.,  9., 10., 11.,  4.,  3.,  2.,  1.]]))




```python
# 张量判断
X == Y
```




    tensor([[False,  True, False,  True],
            [False, False, False, False],
            [False, False, False, False]])




```python
X.sum()

```




    tensor(66.)




```python
a = torch.arange(3).reshape((3, 1))
b = torch.arange(2).reshape((1, 2))
a, b
```




    (tensor([[0],
             [1],
             [2]]),
     tensor([[0, 1]]))




```python
# 矩阵广播
a + b
```




    tensor([[0, 1],
            [1, 2],
            [2, 3]])




```python
X,X[-1], X[1:3]

```




    (tensor([[ 0.,  1.,  2.,  3.],
             [ 4.,  5.,  6.,  7.],
             [ 8.,  9., 10., 11.]]),
     tensor([ 8.,  9., 10., 11.]),
     tensor([[ 4.,  5.,  6.,  7.],
             [ 8.,  9., 10., 11.]]))




```python
X[1, 2] = 9
X
```




    tensor([[ 0.,  1.,  2.,  3.],
            [ 4.,  5.,  9.,  7.],
            [ 8.,  9., 10., 11.]])




```python
X[0:2, :] = 12
X
```




    tensor([[12., 12., 12., 12.],
            [12., 12., 12., 12.],
            [ 8.,  9., 10., 11.]])




```python
# 使用id 
before = id(Y)
Y = Y + X
id(Y) == before
```




    False




```python
# 使用 Z[:] 和 X+= 来节省内存
Z = torch.zeros_like(Y)
print('id(Z):', id(Z))
Z[:] = X + Y
print('id(Z):', id(Z))
```

    id(Z): 5145260816
    id(Z): 5145260816



```python
before = id(X)
X += Y
id(X) == before
```




    True




```python
# numpy 的相互转换
A = X.numpy()
B = torch.tensor(A)
type(A), type(B)
```




    (numpy.ndarray, torch.Tensor)




```python
# 标量
a = torch.tensor([3.5])
a, a.item(), float(a), int(a)
```




    (tensor([3.5000]), 3.5, 3.5, 3)




```python

```


---

> 作者: toxi  
> URL: https://example.com/touch-prefix/  

