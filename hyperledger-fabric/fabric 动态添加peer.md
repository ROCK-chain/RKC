### hyperledger/fabric-samples 中的first-netword 动态添加Org1的peer2
  * cd first-network
    * 启动网络: ./byfn.sh up
  * 复制crypto-config.yaml成crypto-config2.yaml
  * 将crypto-config2.yaml修改Org1的Template.Count由2改为3
  * 执行增加Org1的peer2的相关证书
    ```
    cryptogen extend --config=./crypto-config2.yaml
    ```
  * 新增org1-peer2.yaml
    ``` 
    version: '2'
    
    services:
    
      peer2.org1.example.com:
        container_name: peer2.org1.example.com
        extends:
          file: base/peer-base.yaml
          service: peer-base
        environment:
          - CORE_PEER_ID=peer2.org1.example.com
          - CORE_PEER_ADDRESS=peer2.org1.example.com:11051
          - CORE_PEER_LISTENADDRESS=0.0.0.0:11051
          - CORE_PEER_CHAINCODEADDRESS=peer2.org1.example.com:11052
          - CORE_PEER_CHAINCODELISTENADDRESS=0.0.0.0:11052
          - CORE_PEER_GOSSIP_EXTERNALENDPOINT=peer2.org1.example.com:11051
          - CORE_PEER_LOCALMSPID=Org1MSP
        volumes:
            - /var/run/:/host/var/run/
            - ./crypto-config/peerOrganizations/org1.example.com/peers/peer2.org1.example.com/msp:/etc/hyperledger/fabric/msp
            - ./crypto-config/peerOrganizations/org1.example.com/peers/peer2.org1.example.com/tls:/etc/hyperledger/fabric/tls
        ports:
          - 11051:11051
    
    networks:
      default:
        external:
          name: net_byfn
    ```
  * 运行org1-peer2
    ```
    docker-compose -f org1-peer2.yaml up -d 
    ```
  * 首先进入 cli 容器：
    ```
    docker exec -it cli bash
    ```
    接下来在容器里操作，配置环境变量:
    ``` 
    export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
    export CORE_PEER_ADDRESS=peer2.org1.example.com:11051
    export CORE_PEER_LOCALMSPID="Org1MSP"
    export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer2.org1.example.com/tls/ca.crt
    ```
    加入通道
    ```
    peer channel join -b mychannel.block 
    ```
    安装链码
    ```
    peer chaincode install -n mycc -v 1.0 -l golang -p github.com/chaincode/chaincode_example02/go/ 
    ```
    查询链码
    ```
    peer chaincode query -C mychannel -n mycc -c '{"Args":["query","a"]}'
    ```
    90 验证成功