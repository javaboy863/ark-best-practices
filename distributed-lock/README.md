#现象及背景
下载任务无法执行，一直（几小时内）处于待下载状态。
<div align="left">
<img src="./img/1.png" width="80%" align="center"/>
</div>

#问题
如图，redis分布式锁使用了双重锁，使用不当导致锁无法释放。

<div align="left">
<img src="./img/2.png" width="80%" align="center"/>
</div>

#解法
trylock获取锁，失败则返回不再操作（其他场景根据情况决定是否等待）。原理：默认30S，每10S自动续期时间，直到程序执行完后释放锁。
只有是当前锁持有者才可以释放锁。
<div align="left">
<img src="./img/3.png" width="80%" align="center"/>
</div>