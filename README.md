sudo apt update & sudo apt upgrade -y

sudo apt install curl git wget make jq build-essential pkg-config libssl-dev gcc screen unzip lz4 -y



Install Go
```console
cd $HOME && \
ver="1.21.3" && \
wget "https://golang.org/dl/go$ver.linux-amd64.tar.gz" && \
sudo rm -rf /usr/local/go && \
sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz" && \
rm "go$ver.linux-amd64.tar.gz" && \
echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> $HOME/.bash_profile && \
source $HOME/.bash_profile && \
go version


```
## Install Allorad: Wallet
```console
git clone https://github.com/allora-network/allora-chain.git

cd allora-chain && make all
```

## Install Worker
```console
git clone https://github.com/allora-network/basic-coin-prediction-node

cd basic-coin-prediction-node

mkdir worker-data
mkdir head-data

# Give certain permissions
sudo chmod -R 777 worker-data
sudo chmod -R 777 whead-data

# Create head keys
docker run -it --entrypoint=bash -v ./head-data:/data alloranetwork/allora-inference-base:latest -c "mkdir -p /data/keys && (cd /data/keys && allora-keys)"

# Create worker keys
docker run -it --entrypoint=bash -v ./worker-data:/data alloranetwork/allora-inference-base:latest -c "mkdir -p /data/keys && (cd /data/keys && allora-keys)"

# Copy the key
cat head-data/keys/identity

# Replace head-id with your key
nano docker-compose.yml

# Run worker
docker compose build
docker compose up -d
```
