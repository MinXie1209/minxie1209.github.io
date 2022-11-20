NIO 中, Buffer 抽象了缓冲区的功能

包括:

- 传输数据 : get 和 put 方法

- 标记重置 : 可以通过 mark 标记某个位置,调用 reset 方法重置到该位置

- 私有的成员变量 : (  mark <= position <= limit <= capacity )

  - mark : 标记
  - position : 当前位置
  - limit : 限制的大小
  - capacity : 容量大小

  