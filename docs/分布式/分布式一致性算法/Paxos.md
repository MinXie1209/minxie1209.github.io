### Paxos
Paxos算法分三种角色

- **proposers**-> 提出提案，提案信息包括提案编号和提议的内容

- **acceptor**-> 收到提案后可以接受（accept）提案，若提案获得多数派（majority）的 acceptors 的接受，则称该提案被批准（chosen）；

- **learners**-> 只能“学习”被批准的提案。

> 多数派：指 n / 2 +1 。n为总节点数量

**算法两个阶段：**

**第一阶段：** 
- Proposer选择一个提案编号N,并向半数以上的Acceptor发出Prepare请求。
- 如果Acceptor接受一个提案编号为N的Prepare请求，则返回已经接受的Prepare请求中编号最大的**提案**给Proposer，并且承诺不再接受比N小的提案Prepare请求。

**第二阶段：** 
- 如果Proposer收到半数以上的Acceptor对其发出编号为N的Prepare请求的响应，那么它会向半数以上的Acceptor发起针对提案[N,V]
  的Accept请求，V是第一阶段收到最大提案编号的内容，如果提案内容为空，则V由Proposer自己决定。
- 如果Acceptor收到针对编号提案N的Accept请求，如果Acceptor没有接受比提案编号比N大的Prepare请求，它就接受该提案。

> Proposer可以随时丢弃提案，并且提出新的提案；Acceptor也可以随时响应，接受编号更大的提案。