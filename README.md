# Despliegue de Red de Fármacos - Práctica

Este repositorio contiene la documentación y los pasos para el despliegue de una red de Hyperledger Fabric para el sistema de trazabilidad de fármacos, involucrando tres organizaciones: **Moderno**, **Logistica** y **Delivery**.

Ha sido testado desplegado en Google Cloud con entorno Ubuntu. El ejercicio pretende levantar una red que se ajuste a los siguientes conceptos:

**Tolerancia a fallos:** Utilización de un número impar de Orderers (suelen ser tres) para mantener el consenso de la red operativo.

**Alta disponibilidad de la red:** Al menos dos nodos Peer que contienen la información replicada, garantizando redundancia y disponibilidad de la red.

## Pre-Requisitos

### 1. **Instalar Docker**

```
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo docker run hello-world
```

### 2. **Añadir Grupo Docker y Mi Usuario a ese Grupo (para no usar sudo todo el tiempo)
```
sudo groupadd docker
sudo usermod -aG docker $USER
logout
docker run hello-world
```
### 3. **Instalar Go
```
sudo apt install golang-go
go version
```
### 4. **Instalar NPM
```
sudo apt install npm
```
### 5. **Instalar Docker Compose
```
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
docker-compose --version
```
### 6. **Descargar Repositorios y Binarios (fabric-samples)
```
curl -sSL https://bit.ly/2ysbOFE | bash -s -- 2.3.2 1.5.2
```
### 7. **Descargar Repositorio Práctica
```
git clone https://gitlab.com/STorres17/bsmexecutive.git
mv bsmexecutive practicaEmilio  # Renombrar carpeta principal al gusto
cd practicaEmilio
git checkout master
```
### 8. **Limpiar Imágenes de Docker
```
docker stop $(docker ps -a -q)
docker rm $(docker ps -a -q)
docker volume prune
docker network prune
rm -rf organizations/peerOrganizations
rm -rf organizations/ordererOrganizations
rm -rf channel-artifacts/
mkdir channel-artifacts  # Crear carpeta para artefactos generados
```
## **Levantar los Docker de la CA**
```
docker-compose -f docker/docker-compose-farma-ca.yaml up -d
```
## **Exportar Rutas Necesarias**
```
export PATH=${PWD}/../fabric-samples/bin:${PWD}:$PATH
export FABRIC_CFG_PATH=${PWD}/../fabric-samples/config
```
## **Generar Certificados para Cada Participante**
```
. ./organizations/fabric-ca/registerEnrollFarma.sh && createModerno
. ./organizations/fabric-ca/registerEnrollFarma.sh && createLogistica
. ./organizations/fabric-ca/registerEnrollFarma.sh && createDelivery
. ./organizations/fabric-ca/registerEnrollFarma.sh && createOrderer
```
## **Levantar los Docker de los Peers**
```
docker-compose -f docker/docker-compose-farma.yaml up -d
```
## **Sobrescribir el ```configtx``` y Exportar el PATH**
```
cp configtx/configtxFarma.yaml configtx/configtx.yaml
export FABRIC_CFG_PATH=${PWD}/configtx
```
## **Levantar los 2 Canales**
```
configtxgen -profile FarmaApplicationGenesis -outputBlock ./channel-artifacts/farmaapplicationchannel.block -channelID farmaapplicationchannel
configtxgen -profile VentasGenesis -outputBlock ./channel-artifacts/ventaschannel.block -channelID ventaschannel
```
## **Registrar los Canales en los Orderers**
```
export FABRIC_CFG_PATH=${PWD}/../fabric-samples/config
export ORDERER_CA=${PWD}/organizations/ordererOrganizations/farma.com/orderers/orderer.farma.com/msp/tlscacerts/tlsca.farma.com-cert.pem
export ORDERER_ADMIN_TLS_SIGN_CERT=${PWD}/organizations/ordererOrganizations/farma.com/orderers/orderer.farma.com/tls/server.crt
export ORDERER_ADMIN_TLS
```
## **Añadir los Nodos de Cada Organización a los Canales**

### ** Tres Nodos en el Canal ```farmaapplicationchannel```
```
# Moderno Peer0
export PEER0_MODERNO_CA=${PWD}/organizations/peerOrganizations/moderno.farma.com/peers/peer0.moderno.farma.com/tls/ca.crt
export CORE_PEER_LOCALMSPID="ModernoMSP"
export CORE_PEER_TLS_ROOTCERT_FILE=$PEER0_MODERNO_CA
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/moderno.farma.com/users/Admin@moderno.farma.com/msp
export CORE_PEER_ADDRESS=localhost:7051
peer channel join -b ./channel-artifacts/farmaapplicationchannel.block

# Moderno Peer1
export PEER1_MODERNO_CA=${PWD}/organizations/peerOrganizations/moderno.farma.com/peers/peer1.moderno.farma.com/tls/ca.crt
export CORE_PEER_LOCALMSPID="ModernoMSP"
export CORE_PEER_TLS_ROOTCERT_FILE=$PEER1_MODERNO_CA
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/moderno.farma.com/users/Admin@moderno.farma.com/msp
export CORE_PEER_ADDRESS=localhost:3051
peer channel join -b ./channel-artifacts/farmaapplicationchannel.block

# Logistica Peer0
export PEER0_LOGISTICA_CA=${PWD}/organizations/peerOrganizations/logistica.farma.com/peers/peer0.logistica.farma.com/tls/ca.crt
export CORE_PEER_LOCALMSPID="LogisticaMSP"
export CORE_PEER_TLS_ROOTCERT_FILE=$PEER0_LOGISTICA_CA
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/logistica.farma.com/users/Admin@logistica.farma.com/msp
export CORE_PEER_ADDRESS=localhost:9051
peer channel join -b ./channel-artifacts/farmaapplicationchannel.block

# Delivery Peer0
export PEER0_DELIVERY_CA=${PWD}/organizations/peerOrganizations/delivery.farma.com/peers/peer0.delivery.farma.com/tls/ca.crt
export CORE_PEER_LOCALMSPID="DeliveryMSP"
export CORE_PEER_TLS_ROOTCERT_FILE=$PEER0_DELIVERY_CA
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/delivery.farma.com/users/Admin@delivery.farma.com/msp
export CORE_PEER_ADDRESS=localhost:2051
peer channel join -b ./channel-artifacts/farmaapplicationchannel.block
```

### ** Dos Nodos en el Canal ventaschannel ```ventaschannel```
```
# Moderno Peer0
export PEER0_MODERNO_CA=${PWD}/organizations/peerOrganizations/moderno.farma.com/peers/peer0.moderno.farma.com/tls/ca.crt
export CORE_PEER_LOCALMSPID="ModernoMSP"
export CORE_PEER_TLS_ROOTCERT_FILE=$PEER0_MODERNO_CA
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/moderno.farma.com/users/Admin@moderno.farma.com/msp
export CORE_PEER_ADDRESS=localhost:7051
peer channel join -b ./channel-artifacts/ventaschannel.block

# Moderno Peer1
export PEER1_MODERNO_CA=${PWD}/organizations/peerOrganizations/moderno.farma.com/peers/peer1.moderno.farma.com/tls/ca.crt
export CORE_PEER_LOCALMSPID="ModernoMSP"
export CORE_PEER_TLS_ROOTCERT_FILE=$PEER1_MODERNO_CA
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/moderno.farma.com/users/Admin@moderno.farma.com/msp
export CORE_PEER_ADDRESS=localhost:3051
peer channel join -b ./channel-artifacts/ventaschannel.block

# Delivery Peer0
export PEER0_DELIVERY_CA=${PWD}/organizations/peerOrganizations/delivery.farma.com/peers/peer0.delivery.farma.com/tls/ca.crt
export CORE_PEER_LOCALMSPID="DeliveryMSP"
export CORE_PEER_TLS_ROOTCERT_FILE=$PEER0_DELIVERY_CA
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/delivery.farma.com/users/Admin@delivery.farma.com/msp
export CORE_PEER_ADDRESS=localhost:2051
peer channel join -b ./channel-artifacts/ventaschannel.block
```
## **Instalar Chaincode de Trazabilidad en los Tres Peers y el de Ventas Solo en el Peer Moderno y Peer Delivery**
```
# Empaquetado del código
peer lifecycle chaincode package chaintrazabilidad.tar.gz --path ./chaincodes/farma/trazabilidad/go/ --lang golang --label trazabilidad_1
peer lifecycle chaincode package chainventas.tar.gz --path ./chaincodes/farma/ventas/go/ --lang golang --label ventas_1

# Instalar chaincode en todos los peers
peer lifecycle chaincode install chaintrazabilidad.tar.gz

# Instalar chaincode ventas solo en Moderno y Delivery
peer lifecycle chaincode install chainventas.tar.gz
```

## **Aprobar el Paquete de Chaincode**
```
export NEW_CC_PACKAGE_ID=<ID_CHANICODE>
peer lifecycle chaincode approveformyorg   -o localhost:7050   --ordererTLSHostnameOverride orderer.farma.com   --channelID farmaapplicationchannel   --name trazabilidad_1   --version 1.0   --package-id $NEW_CC_PACKAGE_ID   --sequence 1   --tls   --cafile ${PWD}/organizations/ordererOrganizations/farma.com/orderers/orderer.farma.com/msp/tlscacerts/tlsca.farma.com-cert.pem
```
### **Comitear Chaincode
```
peer lifecycle chaincode commit \
-o localhost:7050 \
--ordererTLSHostnameOverride orderer.farma.com \
--tls \
--cafile ${PWD}/organizations/ordererOrganizations/farma.com/orderers/orderer.farma.com/msp/tlscacerts/tlsca.farma.com-cert.pem \
--channelID farmaapplicationchannel \
--name trazabilidad_1 \
--version 1.0 \
--sequence 1 \
--peerAddresses localhost:7051 \
--tlsRootCertFiles ${PWD}/organizations/peerOrganizations/moderno.farma.com/peers/peer0.moderno.farma.com/tls/ca.crt
```
### **Consultar Commit
```
peer lifecycle chaincode querycommitted --channelID farmaapplicationchannel --name trazabilidad_1 --cafile ${PWD}/organizations/ordererOrganizations/farma.com/orderers/orderer.farma.com/msp/tlscacerts/tlsca.farma.com-cert.pem
peer lifecycle chaincode querycommitted --channelID ventaschannel --name ventas_1 --cafile ${PWD}/organizations/ordererOrganizations/farma.com/orderers/orderer.farma.com/msp/tlscacerts/tlsca.farma.com-cert.pem
```
## **Realizar Transacciones**
### **Ejemplo para Chaincode Trazabilidad
```
# Invocar InitLedger en el Peer Moderno
peer chaincode invoke \
-o localhost:7050 \
--ordererTLSHostnameOverride orderer.farma.com \
--tls \
--cafile ${PWD}/organizations/ordererOrganizations/farma.com/orderers/orderer.farma.com/msp/tlscacerts/tlsca.farma.com-cert.pem \
-C farmaapplicationchannel \
-n trazabilidad_1 \
-c '{"function":"InitLedger","Args":[]}' \
--peerAddresses localhost:7051 \
--tlsRootCertFiles ${PWD}/organizations/peerOrganizations/moderno.farma.com/peers/peer0.moderno.farma.com/tls/ca.crt
```


