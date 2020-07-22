# Lab2 Raft

## 引言

Raft是复制状态机协议，通过将client请求处理位序列（log），并确保所有副机都能按相同顺序指定相同的log。从而确保副机和主机能提供相同的服务。Raft会处理服务器故障，确保恢复故障的服务器其log是同步的。Raft只有在多数服务器存活时才能使用？下面这一段不懂。

> Raft will continue to operate as long as at least a majority of the servers are alive and can talk to each other. If there is no such majority, Raft will make no progress, but will pick up where it left off as soon as a majority can communicate again.