# Python 的动态导入

## 事件起因

公司在做的一款项目，其中有一项需求是将算法模块拆分到各个函数中，然后通过 UI 界面来进行函数的组合，形成可用的工程。开发流程一直是：算法工程师写好算法模块代码，我进行整合修改，将其包装成可供调用的函数，并统一上传至公司内部的 Git 仓库。

有时候函数的组合是比较难以实现高效结果的，算法工程师想要自己写出一套完整可用的工程，可以方便调用。但因为其种类比较繁多，也可能随时被弃用。出于便捷开发的角度，我想要以动态导入的形式来实现。

具体的方案是：

1. 算法工程师自己编写好一个包，包的入口函数则是接受一个需要的字典型数据，同是返回一个字典型的数据。

1. 算法工程师需要复用原项目下已存在包时，则在 `sys.path[0]` 插入所需包的目录，导入需求模块后进行删除。

    ```python
    import sys
    sys.path.insert(0, "/home/guest/projects/another_package")

    from another_sub_package import another_function

    sys.path.pop(0)
    ```

1. 封装好的包，将入口函数于 `__init__.py` 中导入，并放在原项目的指定目录下。

1. 我这边则是编写一个路由函数，根据 URL 中 path 的不同，而进行不同函数的调用。实现的代码如下。

    ```python
    import importlib

    def call_plugin_function(
        request: Request,
        module_name: str,
        service_name: str,
    ) -> Dict[str, Any]:
        module = importlib.import_module(
            "package.{module_name}".format(
                module_name=module_name
            )
        )
        function = getattr(module, function_name)
        parameters = request.json()
        response = function(parameters)
        return response
    ```

## 主要技术实现分析

为什么选择这样的方式呢？在此就对一些方面进行说明下。

1. `sys.path`，Python 官方文档的解释是：

    > 一个由字符串组成的列表，用于指定模块的搜索路径。初始化自环境变量 `PYTHONPATH`，再加上一条与安装有关的默认路径。</br>
    > 程序启动时将初始化本列表，列表的第一项 `path[0]` 目录含有调用 Python 解释器的脚本。如果脚本目录不可用（比如以交互方式调用了解释器，或脚本是从标准输入中读取的），则 `path[0]` 为空字符串，这将导致 Python 优先搜索当前目录中的模块。注意，脚本目录将插入在 `PYTHONPATH` 的条目*之前*。</br>
    > 程序可以随意修改本列表用于自己的目的。只能向 `sys.path` 中添加 `string` 和 `bytes` 类型，其他数据类型将在导入期间被忽略。

    正是如此，我们可以在 `sys.path` 中添加我们指定的目录，来实现外部模块的导入，而不是必须使用原始的目录来进行模块查找。但是也有不好的方面，使用 `sys.path` 导入的包，IDE 并不会有模块检索和提示，需要对原始模块结构比较了解才行。

1. `importlib`，官方文档也是有说明，这个包的目的有两个。

    > 第一个目的是在 Python 源代码中提供 `import` 语句的实现（并且因此而扩展 `__import__()` 函数）。 这提供了一个可移植到任何 Python 解释器的 `import` 实现。 相比使用 Python 以外的编程语言实现方式，这一实现更加易于理解。</br>
    > 第二个目的是实现 `import` 的部分被公开在这个包中，使得用户更容易创建他们自己的自定义对象 (通常被称为 `importer`) 来参与到导入过程中。

    通过文档的说明，可以很清楚地了解到，`importlib.__import__()` 函数就是 `import` 语句的实现。但是 `import` 也是基于 `sys.path` 进行包的检索的，这也就决定了其导入模块的限制。

## 后续产生的问题及探测思路

一开始并没有多想，只是依赖自己对 Python 标准库的记忆，选择了以上的实现方式，项目也是一直处于可使用的状态。但是某次算法工程师进行工程更新时，问题产生了。

问题的主要表现为，算法工程师添加了一个新的包，但是在通过 HTTP 请求调用新的包时，却是得到了之前已经部署过的包的执行结果。

在检查了他的代码后，发现了一个可疑点，那就是他这两个包内，使用 `sys.path.insert()` 导入的包，其内部结果是一致的。目录结果如下所示。

```txt
plugins
├── __init__.py
├── first_project
|   ├── __init__.py
|   └── function.py
└── second_project
    ├── __init__.py
    └── function.py

additional_1
└── component_1
    ├── __init__.py
    └── functions.py

additional_2
└── component_2
    ├── __init__.py
    └── functions.py
```

1. `first_project` 的外部导入方面，写法如下：

    ```python
    import sys
    sys.path.insert(0, r"additional_1/component_1")
    from functions import available_func
    sys.path.pop(0)
    ```

1. `second_project` 的外部导入写法则是：

    ```python
    import sys
    sys.path.insert(0, r"additional_2/component_2")
    from functions import available_func
    sys.path.pop(0)
    ```

在查阅了 Python 官方文档中对于 [导入系统](https://docs.python.org/zh-cn/3/reference/import.html) 一节的说明，了解到了几点事实。

1. 模块检索

    > 为了开始搜索，Python 需要被导入模块（或者包，对于当前讨论来说两者没有差别）的完整限定名称。 此名称可以来自 `import` 语句所带的各种参数，或者来自传给 `importlib.import_module()` 或 `__import__()` 函数的形参。</br>
    > 此名称会在导入搜索的各个阶段被使用，它也可以是指向一个子模块的带点号路径，例如 `foo.bar.baz`。 在这种情况下，Python 会先尝试导入 `foo`，然后是 `foo.bar`，最后是 `foo.bar.baz`。 如果这些导入中的任何一个失败，都会引发 `ModuleNotFoundError`。

1. 模块缓存

    > 在导入搜索期间首先会被检查的地方是 `sys.modules`。这个映射起到缓存之前导入的所有模块的作用（包括其中间路径）。因此如果之前导入过 `foo.bar.baz`，则 `sys.modules` 将包含 `foo`, `foo.bar` 和 `foo.bar.baz` 条目。 每个键的值就是相应的模块对象。

1. `sys.module`

    > `sys.module` 是一个字典，将模块名称映射到已加载的模块。可以操作该字典来强制重新加载模块，或是实现其他技巧。但是，替换的字典不一定会按预期工作，并且从字典中删除必要的项目可能会导致 Python 崩溃。

接下来就是开始分析论证了。我编写了个一个脚本，顺序调用 `first_project` 和 `second_project`，并输出了调用前后的 `sys.modules` 这个字典。脚本大致如下：

```python
def func(module_name: str, function_name: str) -> None:
    print(sys.modules)
    module = importlib.import_module("plugins.{}".format(module_name))
    function = getattr(module, function_name)
    function()
    print(sys.modules)

func("first_project", "first_function")
func("second_project", "second_function")
```

输出结果则是：

```python
# 省略部分库
# 第一次输出
{...}

# 第二次输出

{..., 'functions': <module 'function' from 'additional_1/component_1/functions.py'>,
'plugins.first_project': ..., 'plugins.first_project.first_function': ...}

# 第三次输出和第二次输出一致

# 第四次输出
{..., 'functions': <module 'function' from 'additional_1/component_1/functions.py'>,
'plugins.first_project': ..., 'plugins.first_project.first_function': ...，
'plugins.second_project': ..., 'plugins.second_project.second_function': ...，}
```

好了，问题的产生诱因已经很明确了，那就是 `functions` 模块没有按照我们的想法自动更新。那又是为什么这样呢？

其实刚才对于 **导入系统** 进行部分介绍的时候，就能看出问题所在了，那就是 `sys.modules` 的缓存机制。

1. 我们在第一次使用 `sys.path` 进行指定路径时，改变了系统包检索的结构，得出了 `functions` 这个模块与其找到的路径。
1. 第二次使用 `sys.path` 进行指定路径检索时，因为其内部结构一致，也是名为 `functions` 的模块。但是在 `sys.modules` 中已经拥有该键名，所以不会再次进行导入替换。
1. 因为还在使用原始的 `functions` 所以得到的函数还是原始的函数。（此时如果你把 `additional_2/component_2/functions.py` 中的 `available_func` 进行改名，则会抛出*导入错误*。）

## 解决方案

问题产生的诱因已经找到了，那自然是需要解决方案了。

当然最简单的方法就是规范外部导入包，使用不同的模块名，但是这样有点治标不治本。究其根本，就是 `sys.modules` 缓存了一部分同名模块，那只要使用前后进行模块的重置就好啦。于是对业务代码进行了部分变动，得到了以下的样式：

```python
# 省略其他包的导入
import importlib
import sys

def call_plugin_function(
    request: Request,
    module_name: str,
    service_name: str,
) -> Dict[str, Any]:
    origin_modules = [module for module in sys.modules.keys()]
    try:
        module = importlib.import_module(
            "package.{module_name}".format(
                module_name=module_name
            )
        )
        function = getattr(module, function_name)
        parameters = request.json()
        response = function(parameters)
    finally:
        current_modules = [module for module in sys.modules.keys()]
        for module in current_modules:
            if module not in origin_modules:
                del sys.modules[module]
    return response
```

实现方法就是很简单的删除了动态导入的包，回归了原始库状态。
