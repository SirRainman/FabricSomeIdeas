# 4 LeaderElectionService 接口

gossip\election\election.go

```go
// LeaderElectionService is the object that runs the leader election algorithm
type LeaderElectionService interface {
   // IsLeader returns whether this peer is a leader or not
   IsLeader() bool

   // Stop stops the LeaderElectionService
   Stop()

   // Yield relinquishes the leadership until a new leader is elected,
   // or a timeout expires
   Yield()
}
```

