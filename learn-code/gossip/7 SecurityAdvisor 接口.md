# 7 SecurityAdvisor 接口

gossip\api\channel.go

```go
// SecurityAdvisor defines an external auxiliary object
// that provides security and identity related capabilities
type SecurityAdvisor interface {
   // OrgByPeerIdentity returns the OrgIdentityType
   // of a given peer identity.
   // If any error occurs, nil is returned.
   // This method does not validate peerIdentity.
   // This validation is supposed to be done appropriately during the execution flow.
   OrgByPeerIdentity(PeerIdentityType) OrgIdentityType
}
```

# 