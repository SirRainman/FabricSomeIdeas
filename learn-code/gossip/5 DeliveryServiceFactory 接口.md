# 5 DeliveryServiceFactory 接口

```go
// DeliveryServiceFactory factory to create and initialize delivery service instance
type DeliveryServiceFactory interface {
   // Returns an instance of delivery client
   Service(g GossipServiceAdapter, ordererSource *orderers.ConnectionSource, msc api.MessageCryptoService, isStaticLead bool) deliverservice.DeliverService
}
```

# 