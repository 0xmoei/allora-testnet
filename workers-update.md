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
* Replace `WalletName` & `Mnemonic`
```
{
    "wallet": {
        "addressKeyName": "WalletName",
        "addressRestoreMnemonic": "Mnemonic",
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

### 3- Config App.py
* Register Coingecko https://www.coingecko.com/en/developers/dashboard & Create API KEY
```console
sudo rm -rf app.py && sudo nano app.py
```
* Paste below code
* Replace API your COINGECKO API with
```console
from flask import Flask, Response
import requests
import json
import pandas as pd
import torch
from chronos import ChronosPipeline
 
# create our Flask app
app = Flask(__name__)
 
# define the Hugging Face model we will use
model_name = "amazon/chronos-t5-tiny"
 
# define our endpoint
@app.route("/inference/<string:token>")
def get_inference(token):
    """Generate inference for given token."""
    if not token or token != "ETH":
        error_msg = "Token is required" if not token else "Token not supported"
        return Response(json.dumps({"error": error_msg}), status=400, mimetype='application/json')
    try:
        # use a pipeline as a high-level helper
        pipeline = ChronosPipeline.from_pretrained(
            model_name,
            device_map="auto",
            torch_dtype=torch.bfloat16,
        )
    except Exception as e:
        return Response(json.dumps({"pipeline error": str(e)}), status=500, mimetype='application/json')
 
    # get the data from Coingecko
    # here we'll use last 30 days of ETH/USD
    url = "https://api.coingecko.com/api/v3/coins/ethereum/market_chart?vs_currency=usd&days=30&interval=daily"
 
    headers = {
        "accept": "application/json",
        "x-cg-demo-api-key": "CG-XXXXXXXXXXXXXXXXXXX" # replace with your API key
    }
 
    response = requests.get(url, headers=headers)
    if response.status_code == 200:
        data = response.json()
        df = pd.DataFrame(data["prices"])
        df.columns = ["date", "price"]
        df["date"] = pd.to_datetime(df["date"], unit = "ms")
        df = df[:-1] # removing today's price
        print(df.tail(5))
    else:
        return Response(json.dumps({"Failed to retrieve data from the API": str(response.text)}), 
                        status=response.status_code, 
                        mimetype='application/json')
 
    # define the context and the prediction length
    context = torch.tensor(df["price"])
    prediction_length = 1
 
    try:
        forecast = pipeline.predict(context, prediction_length)  # shape [num_series, num_samples, prediction_length]
        print(forecast[0].mean().item()) # taking the mean of the forecasted prediction
        return Response(str(forecast[0].mean().item()), status=200)
    except Exception as e:
        return Response(json.dumps({"error": str(e)}), status=500, mimetype='application/json')
 
# run our Flask app
if __name__ == '__main__':
    app.run(host="0.0.0.0", port=8000, debug=True)
```

### 4- Config main.py
```
sudo rm -rf main.py && sudo nano main.py
```
* Paste below code
```
import requests
import sys
import json
 
def process(argument):
    headers = {'Content-Type': 'application/json'}
    url = f"http://inference:8000/inference/{argument}"
    response = requests.get(url, headers=headers)
    return response.text
 
if __name__ == "__main__":
    # Your code logic with the parsed argument goes here
    try:
        if len(sys.argv) < 5:
            value = json.dumps({"error": f"Not enough arguments provided: {len(sys.argv)}, expected 4 arguments: topic_id, blockHeight, blockHeightEval, default_arg"})
        else:
            topic_id = sys.argv[1]
            blockHeight = sys.argv[2]
            blockHeightEval = sys.argv[3]
            default_arg = sys.argv[4]
            
            response_inference = process(argument=default_arg)
            response_dict = {"infererValue": response_inference}
            value = json.dumps(response_dict)
    except Exception as e:
        value = json.dumps({"error": {str(e)}})
    print(value)
```

### 5- Config Requirement.txt
```console
sudo rm -rf requirements.txt && sudo nano requirements.txt
```
* Paste below code
```
flask[async]
gunicorn[gthread]
transformers[torch]
pandas
git+https://github.com/amazon-science/chronos-forecasting.git
```

### 6- Run Worker
```console
chmod +x init.config
./init.config
```
* If you need to make changes to your `config.json` , you must rerun this command again after your changes are done


```console
docker compose up -d --build
```

### 7- Check logs
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

