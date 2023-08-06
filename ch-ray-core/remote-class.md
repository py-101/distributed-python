# 分布式类
:label:`remote-class`

:numref:`remote-function` 展示了如何将一个无状态的函数扩展到 Ray 集群上进行分布式计算，但实际的场景中，我们经常需要进行有状态的计算。最简单的有状态计算包括维护一个计数器，每遇到某种条件，计数器加一。这类有状态的计算对于给定的输入，不一定得到确定的输出。单机场景我们可以使用 Python 的类（Class）来实现，计数器可作为类的成员变量。Ray 可以将 Python 类拓展到集群上，即远程类（Remote Class），又被称为行动者（Actor）。Actor 的名字来自 Actor 编程模型 :cite:`hewitt1973Universal`，这是一个典型的分布式计算编程模型，被广泛应用在大数据和人工智能领域，但 Actor 编程模型比较抽象，我们先从计数器的案例来入手。

### 案例1：分布式计数器

```{.python .input}
# Hide code
# Hide outputs
import logging
import ray

if ray.is_initialized:
    ray.shutdown()
ray.init(logging_level=logging.ERROR)
```

Ray 的 Remote Class 也使用 `ray.remote()`

```{.python .input}
@ray.remote
class Counter:
    def __init__(self):
        self.value = 0

    def increment(self):
        self.value += 1
        return self.value

    def get_counter(self):
        return self.value
```

使用 Ray 创建一个 `Counter` 类的分布式实例，或者说创建一个 Actor，需要在类名 `Counter` 后面加上 `remote()`。这样创建的类就是一个分布式的 Actor。

```{.python .input}
counter = Counter.remote()
```

接下来我们要使用 `Counter` 类的计数功能：`increment()` 方法，我们也要在方法后面添加 `remote()`。

```{.python .input}
obj_ref = counter.increment.remote()
print(ray.get(obj_ref))
```

我们可以用同一个类创建不同的 Actor 实例，不同 Actor 之间的成员函数调用可以被并行化执行的，但同一个 Actor 的成员函数调用是顺序执行的。

```{.python .input}
# 创建 10 个 Actor 实例
counters = [Counter.remote() for _ in range(10)]

# 对每个 Actor 进行 increment 操作
# 这些操作可以分布式执行
results = ray.get([c.increment.remote() for c in counters])
print(results)
```

同一个 Actor 实例是互相共享状态的，所谓共享状态是指，Actor 可能被分布式地调度，无论调度到哪个计算节点，对 Actor 实例的任何操作都像对单机 Python 类和实例的操作一样，对象实例的成员变量的数据是可被访问、修改以及实时更新的。

```{.python .input}
# 对第一个 Actor 进行5次 increment 操作
# 这5次 increment 操作是顺序执行的，5次操作共享状态数据 `value`
results = ray.get([counters[0].increment.remote() for _ in range(5)])
print(results)
```

```{.python .input}
# Hide code
ray.shutdown()
```