# 10 PrivdataConfig 结构体

gossip\privdata\config.go

```go
// PrivdataConfig is the struct that defines the Gossip Privdata configurations.
type PrivdataConfig struct {
	// ReconcileSleepInterval determines the time reconciler sleeps from end of an interation until the beginning of the next
	// reconciliation iteration.
	ReconcileSleepInterval time.Duration
	// ReconcileBatchSize determines the maximum batch size of missing private data that will be reconciled in a single iteration.
	ReconcileBatchSize int
	// ReconciliationEnabled is a flag that indicates whether private data reconciliation is enabled or not.
	ReconciliationEnabled bool
	// ImplicitCollectionDisseminationPolicy specifies the dissemination  policy for the peer's own implicit collection.
	ImplicitCollDisseminationPolicy ImplicitCollectionDisseminationPolicy
}
```

