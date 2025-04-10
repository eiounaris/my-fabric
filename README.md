## 生成证书和MSP目录

```bash
# 生成Orderer MSP
cryptogen generate --config=./organizations/cryptogen/crypto-config-orderer.yaml --output="organizations"

# 生成Org0 MSP
cryptogen generate --config=./organizations/cryptogen/crypto-config-org0.yaml --output="organizations"

# 生成Org1 MSP
cryptogen generate --config=./organizations/cryptogen/crypto-config-org1.yaml --output="organizations"

# 复制证书材料到其余节点
scp -r organizations/ordererOrganizations organizations/peerOrganizations futi@192.168.128.51:/home/futi/my-fabric/organizations/

```

## 启动Fabric节点

```bash
# 启动 orderer0 和  peer0-peer4 of org0
docker compose \
  -f compose/docker-compose-orderer0.yaml \
  -f compose/docker-compose-org0-peer0.yaml \
  -f compose/docker-compose-org0-peer1.yaml \
  -f compose/docker-compose-org0-peer2.yaml \
  -f compose/docker-compose-org0-peer3.yaml \
  -f compose/docker-compose-org0-peer4.yaml up

# 启动 orderer1 和  peer0-peer4 of org1
docker compose \
  -f compose/docker-compose-orderer1.yaml \
  -f compose/docker-compose-org1-peer0.yaml \
  -f compose/docker-compose-org1-peer1.yaml \
  -f compose/docker-compose-org1-peer2.yaml \
  -f compose/docker-compose-org1-peer3.yaml \
  -f compose/docker-compose-org1-peer4.yaml up

```

## 导入全局环境变量

```bash
# 导入 configtx.yaml 所在目录
export FABRIC_CFG_PATH=${PWD}/config

# 开启TLS认证
export CORE_PEER_TLS_ENABLED=true

# 组织根证书
export ORDERER_CA=${PWD}/organizations/ordererOrganizations/example.com/tlsca/tlsca.example.com-cert.pem
export PEER0_ORG0_CA=${PWD}/organizations/peerOrganizations/org0.example.com/tlsca/tlsca.org0.example.com-cert.pem
export PEER0_ORG1_CA=${PWD}/organizations/peerOrganizations/org1.example.com/tlsca/tlsca.org1.example.com-cert.pem

# 通道
export CHANNEL_PROFILE=ChannelUsingRaft
export CHANNEL_NAME=mychannel
export BLOCKFILE=${PWD}/channel-artifacts/${CHANNEL_NAME}.block

# 链码
export CC_NAME=mychaincode
export CC_SRC_PATH=asset-transfer-basic/chaincode-go/
export CC_RUNTIME_LANGUAGE=golang
export CC_VERSION=1.0
export CC_SEQUENCE=1

```

## 设置DNS

```bash
sudo vim /etc/hosts

# orderer
192.168.128.50 orderer0.example.com
192.168.128.51 orderer1.example.com
# peer of org0
192.168.128.50 peer0.org0.example.com
192.168.128.50 peer1.org0.example.com
192.168.128.50 peer2.org0.example.com
192.168.128.50 peer3.org0.example.com
192.168.128.50 peer4.org0.example.com
# peer of org1
192.168.128.51 peer0.org1.example.com
192.168.128.51 peer1.org1.example.com
192.168.128.51 peer2.org1.example.com
192.168.128.51 peer3.org1.example.com
192.168.128.51 peer4.org1.example.com
```



## 创建通道，节点加入通道

```bash
# 创建通道创世区块
mkdir channel-artifacts

configtxgen -profile ${CHANNEL_PROFILE} -outputBlock ${BLOCKFILE} -channelID ${CHANNEL_NAME}

scp -r channel-artifacts/ futi@192.168.128.51:/home/futi/my-fabric


# orderer0 加入通道
export ORDERER0_ADMIN_TLS_SIGN_CERT=${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer0.example.com/tls/server.crt
export ORDERER0_ADMIN_TLS_PRIVATE_KEY=${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer0.example.com/tls/server.key
osnadmin channel join --channelID ${CHANNEL_NAME} --config-block ${BLOCKFILE} -o orderer0.example.com:17050 --ca-file ${ORDERER_CA} --client-cert ${ORDERER0_ADMIN_TLS_SIGN_CERT} --client-key ${ORDERER0_ADMIN_TLS_PRIVATE_KEY}

# orderer1 加入通道
export ORDERER1_ADMIN_TLS_SIGN_CERT=${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer1.example.com/tls/server.crt
export ORDERER1_ADMIN_TLS_PRIVATE_KEY=${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer1.example.com/tls/server.key
osnadmin channel join --channelID ${CHANNEL_NAME} --config-block ${BLOCKFILE} -o orderer1.example.com:17051 --ca-file ${ORDERER_CA} --client-cert ${ORDERER1_ADMIN_TLS_SIGN_CERT} --client-key ${ORDERER1_ADMIN_TLS_PRIVATE_KEY}


# org0 加入通道
export CORE_PEER_LOCALMSPID=Org0MSP
export CORE_PEER_TLS_ROOTCERT_FILE=${PEER0_ORG0_CA}
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/org0.example.com/users/Admin@org0.example.com/msp
export CORE_PEER_ADDRESS=peer0.org0.example.com:8050
peer channel join -b ${BLOCKFILE}

# org1 加入通道
export CORE_PEER_LOCALMSPID=Org1MSP
export CORE_PEER_TLS_ROOTCERT_FILE=$PEER0_ORG1_CA
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
export CORE_PEER_ADDRESS=peer0.org1.example.com:9050
peer channel join -b ${BLOCKFILE}

```

## 创建 Org 组织锚节点

```bash
###！！！ (若在通道配置文件设置了锚节点，则忽略以下操作)


export CORE_PEER_LOCALMSPID=Org0MSP
export CORE_PEER_TLS_ROOTCERT_FILE=${PEER0_ORG0_CA}
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/org0.example.com/users/Admin@org0.example.com/msp
export CORE_PEER_ADDRESS=peer0.org0.example.com:8050
export HOST="peer0.org0.example.com"
export PORT=8050
export ORIGINAL=${PWD}/channel-artifacts/${CORE_PEER_LOCALMSPID}config.json 
export MODIFIED=${PWD}/channel-artifacts/${CORE_PEER_LOCALMSPID}modified_config.json 
export OUTPUT=${PWD}/channel-artifacts/${CORE_PEER_LOCALMSPID}anchors.tx

peer channel fetch config ./channel-artifacts/config_block.pb -o orderer0.example.com:7050 --ordererTLSHostnameOverride orderer0.example.com -c ${CHANNEL_NAME} --tls --cafile "$ORDERER_CA"

configtxlator proto_decode --input ./channel-artifacts/config_block.pb --type common.Block --output ./channel-artifacts/config_block.json

jq .data.data[0].payload.data.config ./channel-artifacts/config_block.json >"./channel-artifacts/${CORE_PEER_LOCALMSPID}config.json"

jq '.channel_group.groups.Application.groups.'${CORE_PEER_LOCALMSPID}'.values += {"AnchorPeers":{"mod_policy": "Admins","value":{"anchor_peers": [{"host": "'$HOST'","port": '$PORT'}]},"version": "0"}}' ./channel-artifacts/${CORE_PEER_LOCALMSPID}config.json > ./channel-artifacts/${CORE_PEER_LOCALMSPID}modified_config.json

configtxlator proto_encode --input "${ORIGINAL}" --type common.Config --output ./channel-artifacts/original_config.pb

configtxlator proto_encode --input "${MODIFIED}" --type common.Config --output ./channel-artifacts/modified_config.pb

configtxlator compute_update --channel_id "${CHANNEL_NAME}" --original ./channel-artifacts/original_config.pb --updated ./channel-artifacts/modified_config.pb --output ./channel-artifacts/config_update.pb

configtxlator proto_decode --input ./channel-artifacts/config_update.pb --type common.ConfigUpdate --output ./channel-artifacts/config_update.json

echo '{"payload":{"header":{"channel_header":{"channel_id":"'$CHANNEL_NAME'", "type":2}},"data":{"config_update":'$(cat ./channel-artifacts/config_update.json)'}}}' | jq . > ./channel-artifacts/config_update_in_envelope.json

configtxlator proto_encode --input ./channel-artifacts/config_update_in_envelope.json --type common.Envelope --output "${OUTPUT}"

peer channel update -o orderer0.example.com:7050 --ordererTLSHostnameOverride orderer0.example.com -c ${CHANNEL_NAME} -f ./channel-artifacts/${CORE_PEER_LOCALMSPID}anchors.tx --tls --cafile "$ORDERER_CA"

export CORE_PEER_LOCALMSPID=Org1MSP
export CORE_PEER_TLS_ROOTCERT_FILE=$PEER0_ORG1_CA
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
export CORE_PEER_ADDRESS=peer0.org1.example.com:9050
export HOST="peer0.org1.example.com"
export PORT=9050
export ORIGINAL=./channel-artifacts/${CORE_PEER_LOCALMSPID}config.json 
export MODIFIED=./channel-artifacts/${CORE_PEER_LOCALMSPID}modified_config.json 
export OUTPUT=./channel-artifacts/${CORE_PEER_LOCALMSPID}anchors.tx

peer channel fetch config ./channel-artifacts/config_block.pb -o orderer1.example.com:7050 --ordererTLSHostnameOverride orderer1.example.com -c ${CHANNEL_NAME} --tls --cafile "$ORDERER_CA"

configtxlator proto_decode --input ./channel-artifacts/config_block.pb --type common.Block --output ./channel-artifacts/config_block.json

jq .data.data[0].payload.data.config ./channel-artifacts/config_block.json >"./channel-artifacts/${CORE_PEER_LOCALMSPID}config.json"

jq '.channel_group.groups.Application.groups.'${CORE_PEER_LOCALMSPID}'.values += {"AnchorPeers":{"mod_policy": "Admins","value":{"anchor_peers": [{"host": "'$HOST'","port": '$PORT'}]},"version": "0"}}' ./channel-artifacts/${CORE_PEER_LOCALMSPID}config.json > ./channel-artifacts/${CORE_PEER_LOCALMSPID}modified_config.json

configtxlator proto_encode --input "${ORIGINAL}" --type common.Config --output ./channel-artifacts/original_config.pb

configtxlator proto_encode --input "${MODIFIED}" --type common.Config --output ./channel-artifacts/modified_config.pb

configtxlator compute_update --channel_id "${CHANNEL_NAME}" --original ./channel-artifacts/original_config.pb --updated ./channel-artifacts/modified_config.pb --output ./channel-artifacts/config_update.pb

configtxlator proto_decode --input ./channel-artifacts/config_update.pb --type common.ConfigUpdate --output ./channel-artifacts/config_update.json

echo '{"payload":{"header":{"channel_header":{"channel_id":"'$CHANNEL_NAME'", "type":2}},"data":{"config_update":'$(cat ./channel-artifacts/config_update.json)'}}}' | jq . > ./channel-artifacts/config_update_in_envelope.json

configtxlator proto_encode --input ./channel-artifacts/config_update_in_envelope.json --type common.Envelope --output "${OUTPUT}"

peer channel update -o orderer1.example.com:7050 --ordererTLSHostnameOverride orderer1.example.com -c ${CHANNEL_NAME} -f ./channel-artifacts/${CORE_PEER_LOCALMSPID}anchors.tx --tls --cafile "$ORDERER_CA"

```

## 部署链码

```bash
# 打包链码
pushd ${CC_SRC_PATH}

GO111MODULE=on go mod vendor

popd

peer lifecycle chaincode package ${CC_NAME}.tar.gz --path ${CC_SRC_PATH} --lang ${CC_RUNTIME_LANGUAGE} --label ${CC_NAME}_${CC_VERSION}

scp -r ${CC_NAME}.tar.gz futi@192.168.128.51:/home/futi/my-fabric

# org0 安装链码
export CORE_PEER_LOCALMSPID=Org0MSP
export CORE_PEER_TLS_ROOTCERT_FILE=${PEER0_ORG0_CA}
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/org0.example.com/users/Admin@org0.example.com/msp
export CORE_PEER_ADDRESS=peer0.org0.example.com:8050
peer lifecycle chaincode install ${CC_NAME}.tar.gz

# org1 安装链码
export CORE_PEER_LOCALMSPID=Org1MSP
export CORE_PEER_TLS_ROOTCERT_FILE=${PEER0_ORG1_CA}
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
export CORE_PEER_ADDRESS=peer0.org1.example.com:9050
peer lifecycle chaincode install ${CC_NAME}.tar.gz


# 链码ID环境变量
export PACKAGE_ID=$(peer lifecycle chaincode calculatepackageid ${CC_NAME}.tar.gz)



export CORE_PEER_LOCALMSPID=Org0MSP
export CORE_PEER_TLS_ROOTCERT_FILE=${PEER0_ORG0_CA}
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/org0.example.com/users/Admin@org0.example.com/msp
export CORE_PEER_ADDRESS=peer0.org0.example.com:8050
peer lifecycle chaincode queryinstalled --output json

export CORE_PEER_LOCALMSPID=Org1MSP
export CORE_PEER_TLS_ROOTCERT_FILE=${PEER0_ORG1_CA}
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
export CORE_PEER_ADDRESS=peer0.org1.example.com:9050
peer lifecycle chaincode queryinstalled --output json



export CORE_PEER_LOCALMSPID=Org0MSP
export CORE_PEER_TLS_ROOTCERT_FILE=${PEER0_ORG0_CA}
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/org0.example.com/users/Admin@org0.example.com/msp
export CORE_PEER_ADDRESS=peer0.org0.example.com:8050
peer lifecycle chaincode approveformyorg -o localhost:7050 --ordererTLSHostnameOverride orderer0.example.com --tls --cafile "$ORDERER_CA" --channelID $CHANNEL_NAME --name ${CC_NAME} --version ${CC_VERSION} --package-id ${PACKAGE_ID} --sequence ${CC_SEQUENCE}

export CORE_PEER_LOCALMSPID=Org1MSP
export CORE_PEER_TLS_ROOTCERT_FILE=${PEER0_ORG1_CA}
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
export CORE_PEER_ADDRESS=peer0.org1.example.com:9050
peer lifecycle chaincode approveformyorg -o localhost:7051 --ordererTLSHostnameOverride orderer1.example.com --tls --cafile "$ORDERER_CA" --channelID ${CHANNEL_NAME} --name ${CC_NAME} --version ${CC_VERSION} --package-id ${PACKAGE_ID} --sequence ${CC_SEQUENCE}




export CORE_PEER_LOCALMSPID=Org0MSP
export CORE_PEER_TLS_ROOTCERT_FILE=${PEER0_ORG0_CA}
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/org0.example.com/users/Admin@org0.example.com/msp
export CORE_PEER_ADDRESS=peer0.org0.example.com:8050
peer lifecycle chaincode checkcommitreadiness --channelID ${CHANNEL_NAME} --name ${CC_NAME} --version ${CC_VERSION} --sequence ${CC_SEQUENCE} --output json

export CORE_PEER_LOCALMSPID=Org1MSP
export CORE_PEER_TLS_ROOTCERT_FILE=$PEER0_ORG1_CA
export CORE_PEER_MSPCONFIGPATH=$PWD/organizations/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
export CORE_PEER_ADDRESS=peer0.org1.example.com:9050
peer lifecycle chaincode checkcommitreadiness --channelID ${CHANNEL_NAME} --name ${CC_NAME} --version ${CC_VERSION} --sequence ${CC_SEQUENCE} --output json



peer lifecycle chaincode commit -o orderer0.example.com:7050 --ordererTLSHostnameOverride orderer0.example.com --tls --cafile "$ORDERER_CA" --channelID ${CHANNEL_NAME} --name ${CC_NAME} --version ${CC_VERSION} --sequence ${CC_SEQUENCE} --peerAddresses peer0.org0.example.com:8050 --tlsRootCertFiles $PEER0_ORG0_CA --peerAddresses peer0.org1.example.com:9050 --tlsRootCertFiles $PEER0_ORG1_CA



export CORE_PEER_LOCALMSPID=Org0MSP
export CORE_PEER_TLS_ROOTCERT_FILE=${PEER0_ORG0_CA}
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/org0.example.com/users/Admin@org0.example.com/msp
export CORE_PEER_ADDRESS=peer0.org0.example.com:8050
peer lifecycle chaincode querycommitted --channelID $CHANNEL_NAME --name ${CC_NAME}


export CORE_PEER_LOCALMSPID=Org1MSP
export CORE_PEER_TLS_ROOTCERT_FILE=$PEER0_ORG1_CA
export CORE_PEER_MSPCONFIGPATH=$PWD/organizations/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
export CORE_PEER_ADDRESS=peer0.org1.example.com:9050
peer lifecycle chaincode querycommitted --channelID $CHANNEL_NAME --name ${CC_NAME}

```

## 调用链码

```bash
export FABRIC_CFG_PATH=$PWD/config/ # set the FABRIC_CFG_PATH to point to the core.yaml file in the fabric-samples repository

# set the environment variables that allow you to operate the peer CLI as Org1
# Environment variables for Org1
# The CORE_PEER_TLS_ROOTCERT_FILE and CORE_PEER_MSPCONFIGPATH environment variables point to the Org1 crypto material in the organizations folder.
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_LOCALMSPID=Org0MSP
export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/organizations/peerOrganizations/org0.example.com/peers/peer0.org0.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/org0.example.com/users/Admin@org0.example.com/msp
export CORE_PEER_ADDRESS=peer0.org0.example.com:8050

# initialize the ledger with assets. (Note the CLI does not access the Fabric Gateway peer, so each endorsing peer must be specified.)
peer chaincode invoke -o orderer0.example.com:7050 --ordererTLSHostnameOverride orderer0.example.com --tls --cafile "${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer0.example.com/msp/tlscacerts/tlsca.example.com-cert.pem" -C ${CHANNEL_NAME} -n ${CC_NAME} --peerAddresses peer0.org0.example.com:8050 --tlsRootCertFiles "${PWD}/organizations/peerOrganizations/org0.example.com/peers/peer0.org0.example.com/tls/ca.crt" --peerAddresses peer0.org1.example.com:9050 --tlsRootCertFiles "${PWD}/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt" -c '{"function":"InitLedger","Args":[]}'

# get the list of assets that were added to your channel ledger
peer chaincode query -C ${CHANNEL_NAME} -n ${CC_NAME} -c '{"Args":["GetAllAssets"]}'

# change the owner of an asset on the ledger
peer chaincode invoke -o orderer0.example.com:7050 --ordererTLSHostnameOverride orderer0.example.com --tls --cafile "${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer0.example.com/msp/tlscacerts/tlsca.example.com-cert.pem" -C ${CHANNEL_NAME} -n ${CC_NAME} --peerAddresses peer0.org0.example.com:8050 --tlsRootCertFiles "${PWD}/organizations/peerOrganizations/org0.example.com/peers/peer0.org0.example.com/tls/ca.crt" --peerAddresses peer0.org1.example.com:9050 --tlsRootCertFiles "${PWD}/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt" -c '{"function":"TransferAsset","Args":["asset6","Christopher"]}'

# endorsement policy for the asset-transfer (basic) chaincode requires the transaction to be signed by Org1 and Org2, the chaincode invoke command needs to target both peer0.org1.example.com and peer0.org2.example.com using the --peerAddresses flag
# TLS is enabled for the network, the command also needs to reference the TLS certificate for each peer using the --tlsRootCertFiles flag.

# Set the following environment variables to operate as Org2:
# Environment variables for Org2

export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_LOCALMSPID=Org1MSP
export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
export CORE_PEER_ADDRESS=peer0.org1.example.com:9050

# query the asset-transfer (basic) chaincode running on peer0.org2.example.com
peer chaincode query -C ${CHANNEL_NAME} -n ${CC_NAME} -c '{"Args":["ReadAsset","asset6"]}'

```

## 关闭 fabric

```bash
# 关闭 fabric（不影响后续再启动）
# orderer0 and org0
docker compose \
  -f compose/docker-compose-orderer0.yaml \
  -f compose/docker-compose-org0-peer0.yaml \
  -f compose/docker-compose-org0-peer1.yaml \
  -f compose/docker-compose-org0-peer2.yaml \
  -f compose/docker-compose-org0-peer3.yaml \
  -f compose/docker-compose-org0-peer4.yaml down

# orderer1 and org1
docker compose \
  -f compose/docker-compose-orderer1.yaml \
  -f compose/docker-compose-org1-peer0.yaml \
  -f compose/docker-compose-org1-peer1.yaml \
  -f compose/docker-compose-org1-peer2.yaml \
  -f compose/docker-compose-org1-peer3.yaml \
  -f compose/docker-compose-org1-peer4.yaml down
  
# 删除 fabric 容器
docker rm -f $(docker ps -aq --filter label=service=hyperledger-fabric)
docker rm -f $(docker ps -aq --filter name='dev-peer*')
docker kill "$(docker ps -q --filter name=ccaas)"

# 删除 fabric 链码镜像
docker image rm -f $(docker images -aq --filter reference='dev-peer*')

# 删除 docker 全部持久数据卷
docker volume rm $(docker volume ls -q)

# 删除配置文件
rm -rf organizations/ordererOrganizations organizations/peerOrganizations channel-artifacts mychaincode.tar.gz
```

