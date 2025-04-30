>基于规则的事件在游戏中发生，需要进行追踪并和所有玩家共享时，信息将通过 GameState 进行存储和同步。这些信息包括：
- 游戏已运行的时间（包括本地玩家加入前的运行时间）。
- 每个个体玩家加入游戏的时间和玩家的当前状态。
- 当前 Game Mode 的基类。
- 游戏是否已开始。

# 概览
GameState 负责启用客户端监控游戏状态。从概念上而言，GameState 应该管理所有已连接客户端已知的信息。它能够追踪游戏层面的属性，如已连接玩家的列表、夺旗游戏中的团队得分、开放世界游戏中已完成的任务等等。
Game State 并非追踪玩家特有内容的最佳之处，因为它们由 PlayerState 更清晰地处理。整体而言，GameState 应该追踪游戏进程中变化的属性。这些属性与所有人皆相关，且所有人可见。GameMode 只存在于服务器上，而 GameState 存在于服务器上且会被复制到所有客户端，保持所有已连接机器的游戏进程更新。GameState继承自AInfo，这样在网络环境中可以Replicated到多个Client上面去

# 关系
![[GameState游戏状态-1.png]]
第一个MatchState和相关的回调就是为了在网络中传播同步游戏的状态使用的（记得GameMode在Client并不存在，但是GameState是存在的，所以可以通过它来复制），第二部分是玩家状态列表，同样的如果在Client1想看到Client2的游戏状态数据，则Client2的PlayerState就必须广播过来，因此GameState把当前Server的PlayerState都收集了过来，方便访问使用。  
关于使用，开发者可以自定义GameState子类来存储本GameMode的运行过程中产生的数据（那些想要replicated的!），如果是GameMode游戏运行的一些数据，又不想要所有的客户端都可以看到，则也可以写在GameMode的成员变量中。重复遍，PlayerState是玩家自己的游戏数据，GameInstance里是程序运行的全局数据。