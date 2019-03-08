# Lua 行为树

## 特点

* 行为树不保存状态，所以多个执行逻辑相同的实例可以共享同一个行为树。
* 实例的执行状态保存在 tick 当中。tick 充当黑板。
* 简单，灵活，占用内存小。
* 性能高，对比 [behaviourtree](https://github.com/tanema/behaviourtree.lua)，同样跑一个行为树，300 万次 tick，lbt 在不开启 log 的情况下耗时 13.599 秒。而 behaviourtree 耗时 18.141 秒。

## 实现细节

如果前后两次 tick 返回的 running 节点不一致，那么就存在前一个 running 节点没有关闭的可能性。这时候需要检查前面的那个节点是否仍然打开，如果是打开则手动关闭。

一个 running A 打断 running B 的时候，如何优雅的关闭 B 节点。
现在的实现，是先 open A/run A，再 close B。而符合逻辑做法应该是先关闭 B，再启动 A。
一个可能的解决办法是：在进入一个 running 节点的时候，并不立即执行真正的逻辑，
而是等待一帧再执行。这样的缺点是显而易见的，AI 反应比较迟钝。慢了一拍。

我们可以通过记录节点的状态。is_open = true。
open 的时候需要往黑板写入 is_open 状态。
close 的时候需要往黑板清除 is_open 状态。

## 扩展节点

每个节点有三个回调函数：

* open(tick)
* run(tick)
* close(tick)

tick 是为 AI 分配的行为树运行实例。每一帧执行一次 tick 实例的 tick 函数。

```lua
local Const = require "behaviour_tree.const"

local mt = {}
mt.__index = mt

function mt:open()
end

function mt:run()
end

function mt:close()
end

local function new(args)
    local obj = {
        name = "node_name",
        args = args,
    }
    setmetatable(obj, mt)
    return obj
end

return new
```

## TODO

目前手动关闭节点的时候只关闭了叶子节点，也就是行为节点。从根节点到叶子节点这条路径上的中间控制节点并没有关闭。这种实现方式有可能对 mem_priority / mem_sequence 这种有状态的节点产生影响。下一次进入这些节点的时候会从上次被打断的地方继续执行。这种处理方式是否合理，依赖于实际使用场景。

另外一种实现方式参考 behaviour3js。方法是关闭上一次打开的而已经不在运行中的节点。要注意一种情况就是上一次的 running 正常结束的时候不要再次关闭。behaviour3js 就有这个问题。

经过与 sgpl 以及 zyg 同学讨论，被打断以后继续执行之前子节点的行为比较难以理解。
而且在一个追击的场景下，被高优先级打断，追击节点 close 的时候会将目标置空，回来的时候如果不做检查就出问题了。当然节点是应该做检查，但是如果是从头执行也不会出现这样的问题。所以最终的决定是还是关闭中间节点。

## Example

```
lua example/test.lua
```
