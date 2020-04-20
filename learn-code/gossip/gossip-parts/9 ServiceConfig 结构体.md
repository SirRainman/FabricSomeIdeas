# 9 ServiceConfig 结构体

gossip\service\config.go

```go
// ServiceConfig is the config struct for gossip services
type ServiceConfig struct {
	// PeerTLSEnabled enables/disables Peer TLS.
	PeerTLSEnabled bool
	// Endpoint which overrides the endpoint the peer publishes to peers in its organization.
	Endpoint              string
	NonBlockingCommitMode bool
	// UseLeaderElection defines whenever peer will initialize dynamic algorithm for "leader" selection.
	UseLeaderElection bool
	// OrgLeader statically defines peer to be an organization "leader".
	OrgLeader bool
	// ElectionStartupGracePeriod is the longest time peer waits for stable membership during leader
	// election startup (unit: second).
	ElectionStartupGracePeriod time.Duration
	// ElectionMembershipSampleInterval is the time interval for gossip membership samples to check its stability (unit: second).
	ElectionMembershipSampleInterval time.Duration
	// ElectionLeaderAliveThreshold is the time passes since last declaration message before peer decides to
	// perform leader election (unit: second).
	ElectionLeaderAliveThreshold time.Duration
	// ElectionLeaderElectionDuration is the time passes since last declaration message before peer decides to perform
	// leader election (unit: second).
	ElectionLeaderElectionDuration time.Duration
	// PvtDataPullRetryThreshold determines the maximum duration of time private data corresponding for
	// a given block.
	PvtDataPullRetryThreshold time.Duration
	// PvtDataPushAckTimeout is the maximum time to wait for the acknoledgement from each peer at private
	// data push at endorsement time.
	PvtDataPushAckTimeout time.Duration
	// BtlPullMargin is the block to live pulling margin, used as a buffer to prevent peer from trying to pull private data
	// from peers that is soon to be purged in next N blocks.
	BtlPullMargin uint64
	// TransientstoreMaxBlockRetention defines the maximum difference between the current ledger's height upon commit,
	// and the private data residing inside the transient store that is guaranteed not to be purged.
	TransientstoreMaxBlockRetention uint64
	// SkipPullingInvalidTransactionsDuringCommit is a flag that indicates whether pulling of invalid
	// transaction's private data from other peers need to be skipped during the commit time and pulled
	// only through reconciler.
	SkipPullingInvalidTransactionsDuringCommit bool
}
```

# 