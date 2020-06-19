### hyperledger/fabric-samples 中的first-netword 动态添加组织Org3
  * cd first-network
    * 启动网络: ./byfn.sh up
  * 联盟链新增组织 Org3 ：./eyfn.sh up
    * 手动构建脚本细节
    * 切换目录：cd org3-artifacts
    * 为Org3生成加密材料
      ``` 
      ../../bin/cryptogen generate --config=./org3-crypto.yaml
      ```
    * 生成Org3的配置材料
      ``` 
      export FABRIC_CFG_PATH=$PWD && ../../bin/configtxgen -printOrg Org3MSP > ../channel-artifacts/org3.json
      ```
    * 将 Orderer 的MSP材料复制到 Org3 的crypto-config目录
      ```
      cd ../ && cp -r crypto-config/ordererOrganizations org3-artifacts/crypto-config/
      ```  
    * 进入cli容器
      ```
      docker exec -it cli bash
      ```
    * 导出环境变量
      ```
      export ORDERER_CA=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem  && export CHANNEL_NAME=mychannel
      ```
    * 将旧版本通道配置块保存到 config_block.pb 
      ```
      peer channel fetch config config_block.pb -o orderer.example.com:7050 -c $CHANNEL_NAME --tls --cafile $ORDERER_CA
      ```
    * 使用 configtxlator 工具将通道配置块解码为JSON格式（可读取和修改），并必须删除与更改无关的所有标头，元数据，创建者签名
      ```
      configtxlator proto_decode --input config_block.pb --type common.Block | jq .data.data[0].payload.data.config > config.json
      ```
      花一点时间在选择的文本编辑器（或浏览器）中打开此文件。值得研究，因为它揭示了底层配置结构和可以进行的其他类型的通道更新
    * 使用 jq 工具将Org3配置定义 org3.json 附加到通道的应用程序组字段，并命名输出 modified_config.json 
      ```
      jq -s '.[0] * {"channel_group":{"groups":{"Application":{"groups": {"Org3MSP":.[1]}}}}}' config.json ./channel-artifacts/org3.json > modified_config.json
      ```
    * 将 config.json 转回二进制文件 config.pb
      ```
      configtxlator proto_encode --input config.json --type common.Config --output config.pb
      ```
    * 将 modified_config.json 转二进制文件 modified_config.pb
      ```
      configtxlator proto_encode --input modified_config.json --type common.Config --output modified_config.pb
      ```
    * 用 configtxlator 用来计算这两个配置二进制文件之间的增量。输出一个名为的新二进制文件 org3_update.pb
      ```
      configtxlator compute_update --channel_id $CHANNEL_NAME --original config.pb --updated modified_config.pb --output org3_update.pb
      ```
    * 将 org3_update.pb 解码为可编辑的JSON格式 org3_update.json
      ```
      configtxlator proto_decode --input org3_update.pb --type common.ConfigUpdate | jq . > org3_update.json
      ```
    * 包装信封消息为 org3_update_in_envelope.json
      ```
      echo '{"payload":{"header":{"channel_header":{"channel_id":"mychannel", "type":2}},"data":{"config_update":'$(cat org3_update.json)'}}}' | jq . > org3_update_in_envelope.json
      ```
    * 将 org3_update_in_envelope.json 转最终二进制文件 org3_update_in_envelope.pb
      ```
      configtxlator proto_encode --input org3_update_in_envelope.json --type common.Envelope --output org3_update_in_envelope.pb
      ```
    * Org1 签名配置
      ```
      peer channel signconfigtx -f org3_update_in_envelope.pb
      ```
    * 导出 Org2 环境变量: 
      ```     
      export CORE_PEER_LOCALMSPID="Org2MSP"
      export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt
      export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp
      export CORE_PEER_ADDRESS=peer0.org2.example.com:7051
      ```
    * Org2管理员签名并提交更新 
      ```
      peer channel update -f org3_update_in_envelope.pb -c $CHANNEL_NAME -o orderer.example.com:7050 --tls --cafile $ORDERER_CA
      ```
    * 配置领导者选举，利用动态领导者选举:
      ``` 
      CORE_PEER_GOSSIP_USELEADERELECTION=true
      CORE_PEER_GOSSIP_ORGLEADER=false
      ```
    * 将 Org3 加入通道
      * 打开一个新的终端，从 first-network 启动 Org3 docker compose 
        ```
        docker-compose -f docker-compose-org3.yaml up -d
        ```
      * 进入 Org3cli 容器
        ```
        docker exec -it Org3cli bash
        ```
      * 导出环境变量
        ```
        export ORDERER_CA=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem && export CHANNEL_NAME=mychannel
        ``` 
      * 加入通道
        ```
        peer channel join -b mychannel.block
        ```     
      * 升级和调用Chaincode
        * 到从Org3 CLI中
          ```
          peer chaincode install -n mycc -v 2.0 -p github.com/chaincode/chaincode_example02/go/
          ```
        * 到从CLI中
          安装 
          ```
          peer chaincode install -n mycc -v 2.0 -p github.com/chaincode/chaincode_example02/go/
          ```
          升级
          ```
          peer chaincode upgrade -o orderer.example.com:7050 --tls $CORE_PEER_TLS_ENABLED --cafile $ORDERER_CA -C $CHANNEL_NAME -n mycc -v 2.0 -c '{"Args":["init","a","90","b","210"]}' -P "OR ('Org1MSP.peer','Org2MSP.peer','Org3MSP.peer')"
          ```
        * 到从Org3 CLI中，查询即可
          ```
          peer chaincode query -C $CHANNEL_NAME -n mycc -c '{"Args":["query","a"]}'
          ```
        * 完成    
  * 如果使用过该 eyfn.sh 脚本，则需要关闭网络。这可以通过：./eyfn.sh down