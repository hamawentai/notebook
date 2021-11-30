# 基本操作

## 创建`Tensor`

```python
import torch

# 未初始化
x = torch.empty(5, 3)

# 随机初始化
x = torch.rand(5, 3)

# 根据数据集创建
x = torch.tensor([5.5, 3])

# 根据现有的Tensor来创建

# 返回的tensor默认具有相同的torch.dtype和torch.device
x = x.new_ones(5, 3, dtype=torch.float64)  
# 指定新的数据类型
x = torch.randn_like(x, dtype=torch.float) 
```

| 函数                              | 功能                      |
| --------------------------------- | ------------------------- |
| Tensor(*sizes)                    | 基础构造函数              |
| tensor(data,)                     | 类似np.array的构造函数    |
| ones(*sizes)                      | 全1Tensor                 |
| zeros(*sizes)                     | 全0Tensor                 |
| eye(*sizes)                       | 对角线为1，其他为0        |
| arange(s,e,step)                  | 从s到e，步长为step        |
| linspace(s,e,steps)               | 从s到e，均匀切分成steps份 |
| rand/randn(*sizes)                | 均匀/标准分布             |
| normal(mean,std)/uniform(from,to) | 正态分布/均匀分布         |
| randperm(m)                       | 随机排列                  |

## 运算

### 加

```python
# 等同于数字运算
y = torch.rand(5, 3)

# 函数
torch.add(x, y)
# 有`_`作用于原数据，无则返回结果（副本）
y.add_(x)
```

### 索引

我们还可以使用类似NumPy的索引操作来访问`Tensor`的一部分，需要注意的是：**索引出来的结果与原数据共享内存，也即修改一个，另一个会跟着修改。**

```python
y = x[0, :]
```

| 函数                            | 功能                                                  |
| ------------------------------- | ----------------------------------------------------- |
| index_select(input, dim, index) | 在指定维度dim上选取，比如选取某些行、某些列           |
| masked_select(input, mask)      | 例子如上，a[a>0]，使用ByteTensor进行选取              |
| nonzero(input)                  | 非0元素的下标                                         |
| gather(input, dim, index)       | 根据index，在dim维度上选取数据，输出的size与index一样 |

### 改变形状

用`view()`来改变`Tensor`的形状：

```python
y = x.view(15)
z = x.view(-1, 5)  # -1所指的维度可以根据其他维度的值推出来
```

**注意`view()`返回的新`Tensor`与源`Tensor`虽然可能有不同的`size`，但是是共享`data`的，也即更改其中的一个，另外一个也会跟着改变。(顾名思义，view仅仅是改变了对这个张量的观察角度，内部数据并未改变)**

所以如果想返回一个真正新的副本（即不共享data内存）该怎么办呢？Pytorch还提供了一个`reshape()`可以改变形状，但是此函数并不能保证返回的是其拷贝，所以不推荐使用。推荐先用`clone`创造一个副本然后再使用`view`。

```python
x_cp = x.clone().view(15)
```

使用`clone`还有一个好处是会被记录在计算图中，即梯度回传到副本时也会传到源`Tensor`。

### 线性代数

| 函数                              | 功能                              |
| --------------------------------- | --------------------------------- |
| trace                             | 对角线元素之和(矩阵的迹)          |
| diag                              | 对角线元素                        |
| triu/tril                         | 矩阵的上三角/下三角，可指定偏移量 |
| mm/bmm                            | 矩阵乘法，batch的矩阵乘法         |
| addmm/addbmm/addmv/addr/baddbmm.. | 矩阵运算                          |
| t                                 | 转置                              |
| dot/cross                         | 内积/外积                         |
| inverse                           | 求逆矩阵                          |
| svd                               | 奇异值分解                        |

### `Tensor`和`NumPy`相互转换

### `Tensor`转`NumPy`

很容易用`numpy()`和`from_numpy()`将`Tensor`和NumPy中的数组相互转换。但是需要注意的一点是： **这两个函数所产生的的`Tensor`和NumPy中的数组共享相同的内存（所以他们之间的转换很快），改变其中一个时另一个也会改变！！！**

> 还有一个常用的将NumPy中的array转换成`Tensor`的方法就是`torch.tensor()`, 需要注意的是，此方法总是会进行数据拷贝（就会消耗更多的时间和空间），所以返回的`Tensor`和原来的数据不再共享内存。

```python
a = torch.ones(5)
b = a.numpy()
```

### `NumPy`转`Tensor`

```python
a = np.ones(5)
b = torch.from_numpy(a)
```

## 自动求梯度

如果将其属性`.requires_grad`设置为`True`，它将开始追踪(track)在其上的所有操作（这样就可以利用链式法则进行梯度传播了）。完成计算后，可以调用`.backward()`来完成所有梯度计算。此`Tensor`的梯度将累积到`.grad`属性中。

```python
x = torch.ones(2, 2, requires_grad=True)
print(x)
print(x.grad_fn)

'''
tensor([[1., 1.],
        [1., 1.]], requires_grad=True)
None
'''

# 做一些运算操作
y = x + 2
print(y)
print(y.grad_fn)

'''
tensor([[3., 3.],
        [3., 3.]], grad_fn=<AddBackward>)
<AddBackward object at 0x1100477b8>

'''
```

如果不想要被继续追踪，可以调用`.detach()`将其从追踪记录中分离出来，这样就可以防止将来的计算被追踪，这样梯度就传不过去了。此外，还可以用`with torch.no_grad()`将不想被追踪的操作代码块包裹起来，这种方法在评估模型的时候很常用，因为在评估模型时，我们并不需要计算可训练参数（`requires_grad=True`）的梯度。

```python
with torch.no_grad():
  some operation
```



`Function`是另外一个很重要的类。`Tensor`和`Function`互相结合就可以构建一个记录有整个计算过程的有向无环图（DAG）。每个`Tensor`都有一个`.grad_fn`属性，该属性即创建该`Tensor`的`Function`, 就是说该`Tensor`是不是通过某些运算得到的，若是，则`grad_fn`返回一个与这些运算相关的对象，否则是None。

```python
a = torch.randn(2, 2) # 缺失情况下默认 requires_grad = False
a = ((a * 3) / (a - 1))
print(a.requires_grad) # False
a.requires_grad_(True)
print(a.requires_grad) # True
b = (a * a).sum()
print(b.grad_fn)
# function记录了运算方法，如下的`SumBackward0`表示求和
'''
False
True
<SumBackward0 object at 0x118f50cc0>
'''
```

### 梯度

因为`out`是一个标量，所以调用`backward()`时不需要指定求导变量：

`x`的梯度：$\frac{d(out)}{d(x)}$​

```python
x = torch.ones(2, 2, requires_grad=True)
'''
tensor([[1., 1.],
        [1., 1.]], requires_grad=True)
'''
y = x + 2
z = y * y * 3
out = z.mean()
out.backward() # 等价于 out.backward(torch.tensor(1.))

print(x.grad)
tensor([[4.5000, 4.5000],
        [4.5000, 4.5000]])
```

grad在反向传播过程中是累加的(accumulated)，这意味着每一次运行反向传播，梯度都会累加之前的梯度，所以一般在反向传播之前需把梯度清零。

```python
# 再来反向传播一次，注意grad是累加的
out2 = x.sum()
out2.backward()
print(x.grad)

out3 = x.sum()
x.grad.data.zero_()
out3.backward()
print(x.grad)

'''
tensor([[5.5000, 5.5000],
        [5.5000, 5.5000]])
tensor([[1., 1.],
        [1., 1.]])
'''
```

> ​	<font color='r'>为什么在`y.backward()`时，如果`y`是标量，则不需要为`backward()`传入任何参数；否则，需要传入一个与`y`同形的`Tensor`?</font> 简单来说就是为了避免向量（甚至更高维张量）对张量求导，而转换成标量对张量求导。举个例子，假设形状为 `m x n` 的矩阵 X 经过运算得到了 `p x q` 的矩阵 Y，Y 又经过运算得到了 `s x t` 的矩阵 Z。那么按照前面讲的规则，dZ/dY 应该是一个 `s x t x p x q` 四维张量，dY/dX 是一个 `p x q x m x n`的四维张量。问题来了，怎样反向传播？怎样将两个四维张量相乘？？？这要怎么乘？？？就算能解决两个四维张量怎么乘的问题，四维和三维的张量又怎么乘？导数的导数又怎么求，这一连串的问题，感觉要疯掉…… 为了避免这个问题，我们**不允许张量对张量求导，只允许标量对张量求导，求导结果是和自变量同形的张量**。所以必要时我们要把张量通过将所有张量的元素加权求和的方式转换为标量，举个例子，假设`y`由自变量`x`计算而来，`w`是和`y`同形的张量，则`y.backward(w)`的含义是：先计算`l = torch.sum(y * w)`，则`l`是个标量，然后求`l`对自变量`x`的导数。 [参考](https://zhuanlan.zhihu.com/p/29923090)



# 线性回归

## Zero

### 随机数据产生

```python
import random
def data_iter(batch_size, features, lables):
    num_example = len(features)
    indices = list(range(num_example))
    random.shuffle(indices)
    for i in range(0, num_example, batch_size):
        j = torch.LongTensor(indices[i: min(i+batch_size, num_example)])
        # 0为数据的维度；j为索引
        yield features.index_select(0, j), labels.index_select(0, j)
   
# 未能随机抽取数据
def data_iter(x,y, batch_size):
    len_data = len(x)
    indices = list(range(len_data))
    random.shuffle(indices)
    len_batch = len_data // batch_size
    for i in range(len_batch):
        start = i*batch_size
        end = min((i+1)*batch_size, len_data)
        x_hat = x[start: end,:]
        y_hat = y[:,start: end]
        yield x_hat, y_hat
```

### 手动求梯度

```python
alpha = 0.01
for x_hat,y_hat in data_iter(x,y,batch_size):
    len_data = len(x_hat)
#     y_hat = y_hat.view(len_data, -1)
    temp = (y_hat - torch.mm(x_hat, w) - b) 
    w += (alpha * (temp* x_hat).sum(dim=0) * (1/len_data)).view(2, -1)
    b += alpha * temp.sum() * (1/len_data)
```

## 利用`torch`

```python
num_epochs = 10
# 使用torch的优化方法，可以自动更新梯度
optimizer = optim.SGD(net.parameters(), lr=.003)
for epoch in range(1, num_epochs + 1):
    for X, y in data_iter:
        output = net(X)
        l = loss(output, y.view(-1, 1))
        optimizer.zero_grad() # 梯度清零，等价于net.zero_grad()
        l.backward()
        optimizer.step()
    print('epoch %d, loss: %f' % (epoch, l.item()))
```

# `SoftMax`分类

## `SoftMax`

```python
class SoftReg(nn.Module):
    def __init__(self):
        super(SoftReg, self).__init__()
        self.fc = nn.Linear(28*28, 10)
    def forward(self, x):
        x = x.view(x.shape[0], -1)
        x = self.fc(x)
        return x
```

输出10个值，代表在每个分类上的权重。

## 训练

```python
net = SoftReg()
# softmax-log-NLLLoss
loss = nn.CrossEntropyLoss()
optimizer = torch.optim.Adam(net.parameters(), lr=0.01)
```

其中交叉熵函数与正常的略有不同。

```python
x,y = torch.tensor([[1.,2,3,4],[1.,2,3,4]]), torch.tensor([3,1])
loss(x,y)
'''
tensor(1.4402)
'''
```

计算顺序为：

$softmax \rightarrow log \rightarrow NLLLoss$

`SoftMax`函数：
$$
O_i = \frac{e^{I_i}}{\sum_j{e^{I_J}}}
$$
交叉熵：
$$
y_i = -\sum_i{\hat y*log(y)}
$$
类似于一般的二分类。

example：

* Step1: 其中$[1,2,3,4]$的$exp$和为：84.79，这两条数据的真实分类是3、1，那么其$O_i$分别为$exp(x[3])、exp(x[1])$​
* Step2：y为$[3,1]$，那么计算交叉熵时实际上是与这两个向量相乘$[0,0,0,1]、[0,1,0,0]$
* Step3: $NLLLoss$​做的操作是，对Step2计算出来的两个交叉熵求均值。

```python
len_train, len_test = len(mnist_train), len(mnist_test)
for e in range(5):
    r_sum = 0
    l_sum = 0
    for x,y in train_iter:
        y_hat = net(x) # 256,10
        l = loss(y_hat, y) # 1
        optimizer.zero_grad()
        l.backward()
        optimizer.step()
        r_sum += (y_hat.argmax(dim=1) == y).sum().item()
        l_sum += l.item()
    train_acc = r_sum / len_train
    test_acc = evaluate_accuracy(test_iter, net)
    print('epoch',e, 'loss',l_sum/len_train,'train_acc', train_acc, 'test_acc',test_acc)
```

# 正则化

## $L_2$正则化

在原来的损失函数的基础上增加其参数向量的$L_2$范数。

$ℓ(w1,w2,b)+\frac{2n}{λ}∥w∥^2,$​​​

```python
 # 对权重参数衰减
 optimizer_w = torch.optim.SGD(params=[net.weight], lr=lr, weight_decay=wd)
```
