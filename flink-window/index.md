# Flink Window


<!--more-->
- Flink 只对时间类型的窗口清除，其他类型的窗口不清除
- 每一个window 都有一个 trigger 和一个function(ProcessWindowFunction, ReduceFunction 或者 AggregateFunction)

## 组成方式

- Window Assigner 指派元素如何进入 window
- Trigger 
- Evictor 从window中 剔除某些元素

## Window Assigner

当我们使用window时，比如countWindow，

```java
text.flatMap(new WordCount.Tokenizer())
    .name("tokenizer")
    .keyBy(value -> value.f0)
    .countWindow(windowSize, slideSize)
    .sum(1)
    .name("counter");
```
实际上就调用了 window， evictor 和 trigger   
其中，调用window方法，就是根据 WindowAssigner 创建一个window

```java
package org.apache.flink.streaming.api.datastream;

@Public
public class KeyedStream<T, KEY> extends DataStream<T> {
    public WindowedStream<T, KEY, GlobalWindow> countWindow(long size, long slide) {
        return window(GlobalWindows.create())
                .evictor(CountEvictor.of(size))
                .trigger(CountTrigger.of(slide));
    }
    @PublicEvolving
    public <W extends Window> WindowedStream<T, KEY, W> window(
            WindowAssigner<? super T, W> assigner) {
        return new WindowedStream<>(this, assigner);
    }
    ... ...

}
```



---

> 作者: toxi  
> URL: https://example.com/flink-window/  

