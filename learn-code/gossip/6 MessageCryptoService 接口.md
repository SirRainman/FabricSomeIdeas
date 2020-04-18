# 6 MessageCryptoService 接口

gossip\api\crypto.go

```go
// MessageCryptoService is the contract between the gossip component and the
// peer's cryptographic layer and is used by the gossip component to verify,
// and authenticate remote peers and data they send, as well as to verify
// received blocks from the ordering service.
type MessageCryptoService interface {
   // GetPKIidOfCert returns the PKI-ID of a peer's identity
   // If any error occurs, the method return nil
   // This method does not validate peerIdentity.
   // This validation is supposed to be done appropriately during the execution flow.
   GetPKIidOfCert(peerIdentity PeerIdentityType) common.PKIidType

   // VerifyBlock returns nil if the block is properly signed, and the claimed seqNum is the
   // sequence number that the block's header contains.
   // else returns error
   VerifyBlock(channelID common.ChannelID, seqNum uint64, block *cb.Block) error

   // Sign signs msg with this peer's signing key and outputs
   // the signature if no error occurred.
   Sign(msg []byte) ([]byte, error)

   // Verify checks that signature is a valid signature of message under a peer's verification key.
   // If the verification succeeded, Verify returns nil meaning no error occurred.
   // If peerIdentity is nil, then the verification fails.
   Verify(peerIdentity PeerIdentityType, signature, message []byte) error

   // VerifyByChannel checks that signature is a valid signature of message
   // under a peer's verification key, but also in the context of a specific channel.
   // If the verification succeeded, Verify returns nil meaning no error occurred.
   // If peerIdentity is nil, then the verification fails.
   VerifyByChannel(channelID common.ChannelID, peerIdentity PeerIdentityType, signature, message []byte) error

   // ValidateIdentity validates the identity of a remote peer.
   // If the identity is invalid, revoked, expired it returns an error.
   // Else, returns nil
   ValidateIdentity(peerIdentity PeerIdentityType) error

   // Expiration returns:
   // - The time when the identity expires, nil
   //   In case it can expire
   // - A zero value time.Time, nil
   //   in case it cannot expire
   // - A zero value, error in case it cannot be
   //   determined if the identity can expire or not
   Expiration(peerIdentity PeerIdentityType) (time.Time, error)
}
```

# 