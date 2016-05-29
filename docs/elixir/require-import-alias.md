## 概述

`use Module`除了在`Module`上调用`__using__`宏外, 不做其他任何事情. `import Module` 会把`Module`模块的所有非私有函数和宏引入到当前模块中, 可以直接通过名字调用函数和宏. `require Module` 允许使用一个模块的宏, 但并不导入他们. 必须通过全名(带名称空间)来引用.

例如:

```elixir
defmodule User do
  defmacro __using__(_opts) do
    IO.puts "use User"
  end

  def get_username() do
    "developerworks"
  end
end
```

```elixir
defmodule Hello do
  use User

  def hi() do
    IO.puts "Hello, #{get_username()}"     # User模块未被导入,该函数不存在
  end
end
```

再来看下面的例子:

```elixir
defmodule User do
  defmacro __using__(_opts) do
    quote do          
      import User     
    end               
  end

  def get_username() do
    "developerworks"
  end
end

defmodule ModB do
  use User

  def hi() do
    IO.puts "Hello, #{get_username()}"     # User模块已导入, 该函数存在
  end
end
```

当你使用`use User`的时候, 它生产了一个`import`语句, 把`User`的函数和宏插入到`Hello`中.

## 一个实际的例子

下面是一个实际的例子, 一个Zlib模块用于压缩和解压数据, 代码可以保存为`zlib.exs`脚本文件方便测试


```elixir
defmodule Zlib do

  @moduledoc """
  Zlib compress and decompress functions
  """

  @doc """
  use Zlib
  """
  defmacro __using__(_opts) do
    quote do
      import Zlib
    end
  end

  def compress(data) do
    z = :zlib.open()
    :zlib.deflateInit(z)
    :zlib.deflate(z, data)
    result = :zlib.deflate(z, << >>, :finish)
    :zlib.deflateEnd(z)
    :erlang.list_to_binary(result)
  end

  def decompress(data, wbits \\ 15) do
    z = :zlib.open()
    :zlib.inflateInit(z, wbits)
    [result|_] = :zlib.inflate(z, data)
    :zlib.inflateEnd(z)
    result
  end

end

require Logger
use Zlib

compressed_data = Zlib.compress("test")
Logger.info "#{inspect compressed_data}"
decompressed_data = Zlib.decompress(compressed_data)

Logger.info "#{decompressed_data}"
```

## 正确的方式

上面的代码实际上运行的时候回出错, 通过我的测试`use Zlib`只能用在模块内部. 不能直接在全局空间中使用, 正确的调用代码如下:

```elixir

defmodule Test do
  @moduledoc """
  压缩和解压缩测试模块.
  """
  def run() do
    require Logger
    # 把Zlib模块中的函数和宏导入到当前模块的作用域中.
    use Zlib
    compressed_data = compress("test")
    Logger.info "#{inspect compressed_data}"
    decompressed_data = decompress(compressed_data)
    Logger.info "#{decompressed_data}"
  end
end

Test.run
```

通过上述实验, 我们在模块Zlib中定义了一个 `__using__` 宏, 正如文章最开始所述, `use Zlib`唯一的作用就是调用目标模块`Zlib`中的`__using__`宏, 在`__using__`宏中我们`import`了自身, 也就是导入了`Zlib`模块中的所有公共(定义为`def`而非`defp`)的函数和宏.

可以实验一下, 如果你在`Zlib`模块中增加一个私有函数(`private_test`), 那么我们`use Zlib`后在模块外部调用`private_test`是会发生如下错误的.

```shell
zlib.exs:33: warning: function private_test/0 is unused
** (CompileError) zlib.exs:53: undefined function private_test/0
    (stdlib) lists.erl:1337: :lists.foreach/2
    zlib.exs:41: (file)
    (elixir) lib/code.ex:363: Code.require_file/2
```

另外对于`import`, 还有`only`, `except`选项, 分别表示只导入指定的函数, 以及排除指定的函数, 例如:

```elixir
import Zlib, only: compress
import Zlib, except: decompress
```

有的模块中包含多个同名, 参数数(arity)不同的函数, 你甚至可以按函数的参数个数来指定要导入的特定函数, 比如, 在Elixir内置的模块`List`中存在`List.to_integer/1`和`List.to_integer/2`两个名为`to_integer`的函数, 如果你只想导入`to_integer/1`,那么可以按下面的方式来导入:

```elixir
import List, only: [to_integer: 1]
```

这里与上面的区别在于`only`接受的是一个列表, 也就是说, 你还可以导入其他你需要指定的函数, 比如下面的代码仅导入了`to_integer/1`和`duplicate/2`两个`List`模块中的函数.

```elixir
import List, only: [to_integer: 1, duplicate: 2]
```

## 最后

最后的`require`比较简单, 其作用类似于C语言中的`#include "zlib.h"`把头文件包含进来, 编译链接的时候就会去查找对应的共享库. `require`的作用与之类似,要使用其他模块所提供的函数和宏, 就必须要`require`进来.
