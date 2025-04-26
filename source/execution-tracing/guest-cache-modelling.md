
# 客户 CPU 缓存建模

Renode 可以对内存访问（由执行跟踪子系统生成）执行事后分析，以模拟 CPU 缓存行为并生成使用情况统计信息。

要从仿真生成内存访问跟踪，请将以下行添加到 `resc` 文件中：

```none
cpu MaximumBlockSize 1
cpu CreateExecutionTracing "tracer" $ORIGIN/trace.log PCAndOpcode
tracer TrackMemoryAccesses
```

```{note}
将 `cpu` 替换为要分析的核心名称（来自 `repl` 文件）。
```

然后，您可以将生成的 `trace.log` 文件传递给 [Renode Cache Modeling Analyzer](https://github.com/renode/renode/tree/master/tools/guest_cache)。请注意，在运行分析器之前，您需要安装 `requirements.txt` 中列出的依赖项。

您可以将分析器与其内置预设一起使用：

```sh
./renode_cache_interface.py trace.log presets 'fu740.u74'
```

预设存储在 [`presets.py`](https://github.com/renode/renode/blob/master/tools/guest_cache/src/presets.py) 文件中。要创建新预设，请添加新条目：

```python
'new_preset': {
    'l1d': Cache(
        name='l1i,name',            # Cache name - used in the `printd` debug helpers.
        cache_width=15,             # number of bits used to address the cache
        block_width=6,              # number of bits used in a cache block
        memory_width=64,            # number of bits used to address the main memory
        lines_per_set=2,            # 2 way associativity
        replacement_policy="FIFO"   # replacement policy
        ),
    'flush_opcodes': {
        0x1000: 'd',                # flush `l1d` when after the `0x1000` opcode is executed 
    },
    'invalidate_on_io': True        # flush the data cache when an MemoryMapped I/O operation is performed
}
```

这将创建仅包含数据缓存的预设。您可以通过添加 `l1i` 对象来添加 Instruction 缓存。有关使用预设进行缓存配置的更多信息，请参阅 [`cache.py`](https://github.com/renode/renode/blob/master/tools/guest_cache/src/cache.py) 文件中的文档。目前，仅支持 1 级指令和数据缓存。

您还可以使用 CLI 参数配置缓存：

```sh
./renode_cache_interface.py trace.log config            \
                          --memory_width 64             \
                          --l1i_cache_width 15          \
                          --l1i_block_width 6           \
                          --l1i_lines_per_set 4         \
                          --l1i_replacement_policy LRU  \
                          --l1d_cache_width 15          \
                          --l1d_block_width 6           \
                          --l1d_lines_per_set 8         \
                          --l1d_replacement_policy LRU
```

有关使用 CLI 进行缓存配置的更多信息，请参阅 `./renode_cache_interface.py trace.log config --help` 命令的输出。

分析的输出示例：

```none
$ ./renode_cache_interface.py trace.log presets fu740.u74
l1i,u74 configuration:
Cache size:          32768 bytes
Block size:          64 bytes
Number of lines:     512
Number of sets:      128 (4 lines per set)
Replacement policy:  RAND

l1d,u74 configuration:
Cache size:          32768 bytes
Block size:          64 bytes
Number of lines:     512
Number of sets:      64 (8 lines per set)
Replacement policy:  RAND

Instructions read: 174620452
Total memory operations: 68952483 (read: 50861775, write 18090708)
Total I/O operations: 1875 (read: 181, write 1694)

l1i,u74 results:
Misses: 168
Hits: 174620284
Invalidations: 3
Hit ratio: 100.0%

l1d,u74
Misses: 17320212
Hits: 51632271
Invalidations: 17319700
Hit ratio: 74.88%
```

还可以通过传递 `--output <filaname>` 标志来生成输出 JSON 文件：

```js
{
   "l1i,u74":{
      "hit":174620284,
      "miss":168,
      "invalidations":3
   },
   "l1d,u74":{
      "hit":51632271,
      "miss":17320212,
      "invalidations":17319700
   }
}
```
