# Update
***Update:*** We are pleased to announce a major testnet upgrade that simplifies worker node deployment, ensuring a smoother and more efficient experience for users engaging with Allora Network's testnet.




***Points:*** We’re also working on retroactively applying points for those who have been running nodes up until now. Additionally, we’ll be adjusting the scale of points received by workers over the next week for those who have been running nodes up to now.
​
Workers that spin up a node using the new process should see points within an hour of running the node.

### If you are new, make sure you installed dependecies [here](https://github.com/0xmoei/allora-testnet/blob/main/README.md#install-dependecies)

### 1- Remove Old files
```console
cd $HOME && cd basic-coin-prediction-node

docker compose down -v

cd $HOME && rm -rf basic-coin-prediction-node
```

### 2- Make sure you have an Allorad wallet & faucet
[Guide](https://github.com/0xmoei/allora-testnet?tab=readme-ov-file#install-allorad-wallet)

Check your wallet name and address
```console
allorad keys list
```
* save wallet name for next steps

#

# 3 Topic Workers using Basic-Price-Prediction model
## Install & Run Worker

### 1- Clone worker
```console
cd $HOME
git clone https://github.com/allora-network/basic-coin-prediction-node
cd basic-coin-prediction-node
```

### 2- Config Worker
```console
# Remove config file
rm -rf config.json

# Create new config file
nano config.json
```

**Paste below code in it**
* Replace your wallet `Seed Phrase`
* `addressKeyName` was set as testkey since we choose it in step: Add Wallet
```
{
    "wallet": {
        "addressKeyName": "testkey",
        "addressRestoreMnemonic": "Seed Phrase",
        "alloraHomeDir": "",
        "gas": "1000000",
        "gasAdjustment": 1.0,
        "nodeRpc": "https://sentries-rpc.testnet-1.testnet.allora.network/",
        "maxRetries": 1,
        "delay": 1,
        "submitTx": false
    },
    "worker": [
        {
            "topicId": 1,
            "inferenceEntrypointName": "api-worker-reputer",
            "loopSeconds": 5,
            "parameters": {
                "InferenceEndpoint": "http://inference:8000/inference/{Token}",
                "Token": "ETH"
            }
        },
        {
            "topicId": 2,
            "inferenceEntrypointName": "api-worker-reputer",
            "loopSeconds": 5,
            "parameters": {
                "InferenceEndpoint": "http://inference:8000/inference/{Token}",
                "Token": "ETH"
            }
        },
        {
            "topicId": 7,
            "inferenceEntrypointName": "api-worker-reputer",
            "loopSeconds": 5,
            "parameters": {
                "InferenceEndpoint": "http://inference:8000/inference/{Token}",
                "Token": "ETH"
            }
        }
    ]
}
```
Ctrl+X+Y+Enter to save & exit

### 3- Run Worker
```console
chmod +x init.config
./init.config
```
* If you need to make changes to your `config.json` , you must rerun this command again after your changes are done


```console
docker compose up -d --build
```

### 4- Check logs
Containers:
```console
docker compose ps
```

worker:
```console
docker compose logs -f worker
```
![image](https://github.com/user-attachments/assets/63ca0e84-c802-416a-a872-af6331aa776f)

inference:
```console
docker compose logs -f inference
```
![image](https://github.com/user-attachments/assets/a8133f85-b643-484d-beeb-cdfb51d6fd5c)

updater:
```console
docker compose logs -f updater

# Response:
# updater-basic-eth-pred  | UPDATING INFERENCE WORKER DATA
# updater-basic-eth-pred  | Response content is '0'
```

# You are going to receive points in the [dashboard](https://app.allora.network/points/leaderboard) in the next few hours
