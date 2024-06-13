pip install allocmd==1.0.4

export WORKER=0xmoei
source $HOME/.bash_profile



allocmd init --name $WORKER --topic 1 --env dev
allocmd init --name moei --topic 1 --env dev


docker-compose -f dev-docker-compose.yaml up --build



