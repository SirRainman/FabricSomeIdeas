# 8 GossipMetrics 结构体 

gossip\metrics\metrics.go

```go
// GossipMetrics encapsulates all of gossip metrics
type GossipMetrics struct {
   StateMetrics      *StateMetrics
   ElectionMetrics   *ElectionMetrics
   CommMetrics       *CommMetrics
   MembershipMetrics *MembershipMetrics
   PrivdataMetrics   *PrivdataMetrics
}
```

# 