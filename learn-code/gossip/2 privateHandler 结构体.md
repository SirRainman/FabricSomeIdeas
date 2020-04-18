# 2 privateHandler 结构体

gossip\service\gossip_service.go

```go
type privateHandler struct {
	support     Support
	coordinator gossipprivdata.Coordinator
	distributor gossipprivdata.PvtDataDistributor
	reconciler  gossipprivdata.PvtDataReconciler
}
```

# 