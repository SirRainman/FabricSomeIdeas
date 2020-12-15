# Kubernetes上使用外部链码

在kubernetes上运行链码容器时遇到的缺点：

1. the dependency of the Docker daemon in the peer leaded to some configurations that might not be acceptable in production environments, 
   1. such as the use of the Docker Socket of the worker nodes in the peer to deploy the chaincode containers, hence making these containers out of Kubernetes domain, 
   2. or the use of Docker-in-Docker (DinD) solutions, which required these containers to be privileged.

# 0 需要准备的

-  **A Kubernetes cluster**. 

- **Hyperledger Fabric 2.3.0 docker images**. When launching the deployments of the Kubernetes yaml descriptors.
- **Hyperledger Fabric 2.2.0 binaries**. We will need them to create the crypto-config and the channel-artifacts.

# 1 具体步骤

## 1.1 安装二进制文件

1. build
2. detect
3. release
4. discover



The definition of the builders can be found inside the org1 folder in the `builders-config.yaml` . 

- This file has all the default options to configure the peer of the `core.yaml` except the *core_chaincode_externalbuilders* option, which has a custom builder configuration like the following.

```yaml
# List of directories to treat as external builders and launchers for
# chaincode. The external builder detection processing will iterate over the
# builders in the order specified below.
            externalBuilders:
              - name: external-builder
                path: /builders/external
                environmentWhitelist:
                   - GOPROXY
```

package the chaincode with some requirements. 

- There has to be a connection.json file that contains the information of the connection to the external chaincode server
- This includes the address, TLS certificates and dial timeout configurations.

```json
{
    "address": "chaincode-marbles-org1.hyperledger:7052",
    "dial_timeout": "10s",
    "tls_required": false,
    "client_auth_required": false,
    "client_key": "-----BEGIN EC PRIVATE KEY----- ... -----END EC PRIVATE KEY-----",
    "client_cert": "-----BEGIN CERTIFICATE----- ... -----END CERTIFICATE-----",
    "root_cert": "-----BEGIN CERTIFICATE---- ... -----END CERTIFICATE-----"
}
```



repackage it again 

- alongside with another file called metadata.json , 
- which includes information about the type of chaincode to be processed, the path where the chaincode resides and the label we want to give to that chaincode.

```json
{"path":"","type":"external","label":"marbles"}
```



# 2 优缺点分析



