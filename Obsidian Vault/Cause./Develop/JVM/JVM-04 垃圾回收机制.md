## 可达性分析

引用计数法：可以用来判断对象被引用的次数，在任何某一具体时刻如果引用次数为0则代表可以被回收。

## 新生代回收

Minor GC触发机制：当年轻代满时就会触发Minor GC，这里的年轻代满指的是Eden代满如果是Survivor满不会引发GC。

## 老年代回收

Full GC触发机制

1. 调用System.gc时系统建议执行Full GC但不是必然执行
2. 老年代空间不足
3. 方法区不足
4. 通过Minor GC后进入老年代的平均大小大于老年代的可用内存
5. 由Eden区、survivor space1（From Space）区向survivor space2（To Space）区复制时对象大小大于To Space可用内存则把该对象转存到老年代，且老年代的可用内存小于该大小
6. 永久代满时也会引发Full GC，会导致Class、Method元信息的卸载