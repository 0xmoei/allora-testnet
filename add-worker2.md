### Make sure you have uALLO faucet in your wallet in order to register 2nd worker

#

### Steps:

```console
cd $HOME && cd basic-coin-prediction-node

docker compose down
docker container stop worker-basic-eth-pred
docker container rm worker-basic-eth-pred

mkdir workers
mkdir workers/worker-1 workers/worker-2

sudo chmod -R 777 workers/worker-1
sudo chmod -R 777 workers/worker-2

sudo docker run -it --entrypoint=bash -v ./workers/worker-1:/data alloranetwork/allora-inference-base:latest -c "mkdir -p /data/keys && (cd /data/keys && allora-keys)"
sudo docker run -it --entrypoint=bash -v ./workers/worker-2:/data alloranetwork/allora-inference-base:latest -c "mkdir -p /data/keys && (cd /data/keys && allora-keys)"

rm -rf docker-compose.yml && nano docker-compose.yml
```

* Copy & Paste the following code in it
* Replace `head-id` & `WALLET_SEED_PHRASE` in worker-1 and worker-2 containers
```console
version: '3'

services:
  inference:
    container_name: inference
    build:
      context: .
    command: python -u /app/app.py
    ports:
      - "8000:8000"
    networks:
      eth-model-local:
        aliases:
          - inference
        ipv4_address: 172.22.0.4
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/inference/ETH"]
      interval: 10s
      timeout: 10s
      retries: 12
    volumes:
      - ./inference-data:/app/data
  
  updater:
    container_name: updater
    build: .
    environment:
      - INFERENCE_API_ADDRESS=http://inference:8000
    command: >
      sh -c "
      while true; do
        python -u /app/update_app.py;
        sleep 24h;
      done
      "
    depends_on:
      inference:
        condition: service_healthy
    networks:
      eth-model-local:
        aliases:
          - updater
        ipv4_address: 172.22.0.5
  
  head:
    container_name: head
    image: alloranetwork/allora-inference-base-head:latest
    environment:
      - HOME=/data
    entrypoint:
      - "/bin/bash"
      - "-c"
      - |
        if [ ! -f /data/keys/priv.bin ]; then
          echo "Generating new private keys..."
          mkdir -p /data/keys
          cd /data/keys
          allora-keys
        fi
        allora-node --role=head --peer-db=/data/peerdb --function-db=/data/function-db  \
          --runtime-path=/app/runtime --runtime-cli=bls-runtime --workspace=/data/workspace \
          --private-key=/data/keys/priv.bin --log-level=debug --port=9010 --rest-api=:6000 \
          --boot-nodes=/dns4/head-0-p2p.v2.testnet.allora.network/tcp/32130/p2p/12D3KooWGKY4z2iNkDMERh5ZD8NBoAX6oWzkDnQboBRGFTpoKNDF
    ports:
      - "6000:6000"
    volumes:
      - ./head-data:/data
    working_dir: /data
    networks:
      eth-model-local:
        aliases:
          - head
        ipv4_address: 172.22.0.100

  worker-1:
    container_name: worker-1
    environment:
      - INFERENCE_API_ADDRESS=http://inference:8000
      - HOME=/data
    build:
      context: .
      dockerfile: Dockerfile_b7s
    entrypoint:
      - "/bin/bash"
      - "-c"
      - |
        if [ ! -f /data/keys/priv.bin ]; then
          echo "Generating new private keys..."
          mkdir -p /data/keys
          cd /data/keys
          allora-keys
        fi
        # Change boot-nodes below to the key advertised by your head
        allora-node --role=worker --peer-db=/data/peerdb --function-db=/data/function-db \
          --runtime-path=/app/runtime --runtime-cli=bls-runtime --workspace=/data/workspace \
          --private-key=/data/keys/priv.bin --log-level=debug --port=9011 \
          --boot-nodes=/ip4/172.22.0.100/tcp/9010/p2p/head-id \
          --topic=allora-topic-1-worker --allora-chain-worker-mode=worker \
          --allora-chain-restore-mnemonic='WALLET_SEED_PHRASE' \
          --allora-node-rpc-address=https://allora-rpc.edgenet.allora.network/ \
          --allora-chain-key-name=worker-1 \
          --allora-chain-topic-id=1
    volumes:
      - ./workers/worker-1:/data
    working_dir: /data
    depends_on:
      - inference
      - head
    networks:
      eth-model-local:
        aliases:
          - worker1
        ipv4_address: 172.22.0.12

  worker-2:
    container_name: worker-2
    environment:
      - INFERENCE_API_ADDRESS=http://inference:8000
      - HOME=/data
    build:
      context: .
      dockerfile: Dockerfile_b7s
    entrypoint:
      - "/bin/bash"
      - "-c"
      - |
        if [ ! -f /data/keys/priv.bin ]; then
          echo "Generating new private keys..."
          mkdir -p /data/keys
          cd /data/keys
          allora-keys
        fi
        # Change boot-nodes below to the key advertised by your head
        allora-node --role=worker --peer-db=/data/peerdb --function-db=/data/function-db \
          --runtime-path=/app/runtime --runtime-cli=bls-runtime --workspace=/data/workspace \
          --private-key=/data/keys/priv.bin --log-level=debug --port=9013 \
          --boot-nodes=/ip4/172.22.0.100/tcp/9010/p2p/head-id \
          --topic=allora-topic-2-worker --allora-chain-worker-mode=worker \
          --allora-chain-restore-mnemonic='WALLET_SEED_PHRASE' \
          --allora-node-rpc-address=https://allora-rpc.edgenet.allora.network/ \
          --allora-chain-key-name=worker-2 \
          --allora-chain-topic-id=2
    volumes:
      - ./workers/worker-2:/data
    working_dir: /data
    depends_on:
      - inference
      - head
    networks:
      eth-model-local:
        aliases:
          - worker1
        ipv4_address: 172.22.0.13
  
networks:
  eth-model-local:
    driver: bridge
    ipam:
      config:
        - subnet: 172.22.0.0/24

volumes:
  inference-data:
  workers:
  head-data:
```
```console
docker compose --build up -d
```

## Check your node status
### Check running docker containers
```console
# Ensure you are in the right directory
cd $HOME && cd basic-coin-prediction-node

# Check worker 2 logs
docker compose logs -f worker-2

# Check worker 1 logs
docker compose logs -f worker-1
```
> You must have `Success: register node Tx Hash` in worker 1 & 2 logs
> Success: register node Tx Hash:=82BF67E2E1247B226B8C5CFCF3E4F41076909ADABF3852C468D087D94BD9FC3B

![Screenshot_80](https://github.com/0xmoei/allora-testnet/assets/90371338/cefe126e-4ecb-4af3-9444-4e5e014fed52)

### Check Worker node:
Check topic 1:
```console
network_height=$(curl -s -X 'GET' 'https://allora-rpc.edgenet.allora.network/abci_info?' -H 'accept: application/json' | jq -r .result.response.last_block_height) && \
curl --location 'http://localhost:6000/api/v1/functions/execute' --header 'Content-Type: application/json' --data '{
    "function_id": "bafybeigpiwl3o73zvvl6dxdqu7zqcub5mhg65jiky2xqb4rdhfmikswzqm",
    "method": "allora-inference-function.wasm",
    "parameters": null,
    "topic": "allora-topic-1-worker",
    "config": {
        "env_vars": [
            {
                "name": "BLS_REQUEST_PATH",
                "value": "/api"
            },
            {
                "name": "ALLORA_ARG_PARAMS",
                "value": "ETH"
            },
            {
                "name": "ALLORA_BLOCK_HEIGHT_CURRENT",
                "value": "'"${network_height}"'"
            }
        ],
        "number_of_nodes": -1,
        "timeout": 2
    }
}' | jq
```

Check topic 2:
```console
network_height=$(curl -s -X 'GET' 'https://allora-rpc.edgenet.allora.network/abci_info?' -H 'accept: application/json' | jq -r .result.response.last_block_height) && \
curl --location 'http://localhost:6000/api/v1/functions/execute' --header 'Content-Type: application/json' --data '{
    "function_id": "bafybeigpiwl3o73zvvl6dxdqu7zqcub5mhg65jiky2xqb4rdhfmikswzqm",
    "method": "allora-inference-function.wasm",
    "parameters": null,
    "topic": "allora-topic-2-worker",
    "config": {
        "env_vars": [
            {
                "name": "BLS_REQUEST_PATH",
                "value": "/api"
            },
            {
                "name": "ALLORA_ARG_PARAMS",
                "value": "ETH"
            },
            {
                "name": "ALLORA_BLOCK_HEIGHT_CURRENT",
                "value": "'"${network_height}"'"
            }
        ],
        "number_of_nodes": -1,
        "timeout": 2
    }
}' | jq
```
Response:
```
{
  "code": "200",
  "request_id": "d9450b57-65d3-45f0-b3a2-201b8f2e4cbc",
  "results": [
    {
      "result": {
        "stdout": "{\"infererValue\": \"2962.3654930667144\"}\n\n",
        "stderr": "",
        "exit_code": 0
      },
      "peers": [
        "12D3KooWFzs8C6Da4MrxU8JqCF4LQW3n6pZYnnWZF5TCqFgxM2Ph"
      ],
      "frequency": 100
    }
  ],
  "cluster": {
    "peers": [
      "12D3KooWFzs8C6Da4MrxU8JqCF4LQW3n6pZYnnWZF5TCqFgxM2Ph"
    ]
  }
}
```

### Check Updater node:
```console
curl http://localhost:8000/update
```
Response:
```
0
```

### Check Inference node:
```console
curl http://localhost:8000/inference/ETH
```
Response:
```
{"value":"2564.021586281073"}
```

### Check Docker containers
```console
docker ps
```
