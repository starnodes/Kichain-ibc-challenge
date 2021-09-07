# testnet-ki-ibc

This guide describes how to set up a repeater for cross translation between two different networks: Ki-chain and Rizon.
In my example, the repeater is installed to the Ki-chain node. A remote connection will be made with the Rizon network node. 

First, make sure that Rizon support IBC
rizond q ibc-transfer params
#receive_enabled: true
#send_enabled: true
Its OK!

Install and configure Relay on the Ki-chain host

Install the repeater

git clone https://github.com/cosmos/relayer.git
cd relayer
make install
cd

rly version
Output:
#version: 1.0.0-rc1-152-g112205b

Initializing the repeater
rly config init

We need to create a folder with network configuration and go to it:
mkdir ~/rly_config && cd ~/rly_config

Create a settings file for both networks 
nano kichain-t4-config.json
{
  "chain-id": "kichain-t-4",
  "rpc-addr": "http://127.0.0.1:26657",
  "account-prefix": "tki",
  "gas-adjustment": 1.5,
  "gas-prices": "0.025utki",
  "trusting-period": "48h"
}

nano umee-betanet-1-config.json

{
"chain-id": "umee-betanet-1",
"rpc-addr": "http://global_ip_adress:26657",
"account-prefix": "umee",
"gas-adjustment": 1.5,
"gas-prices": "0.025uumee",
"trusting-period": "48h"
}

parse these settings into the config of the relay
rly chains add -f umee-betanet-1-config.json
rly chains add -f kichain-t4-config.json
cd

Configure a settings file for Rizon network
nano ~/.umee/config/config.toml
  
We need to change [rpc] section
laddr = "tcp://0.0.0.0:26657"
  
We need to create new wallets:
rly keys add umee-betanet-1 wallet_name
rly keys add kichain-t-4 wallet_name

So you can restore your wallets, use commands:
rly keys restore umee-betanet-1 wallet_name "mnemonic"
rly keys restore kichain-t-4 wallet_name "mnemonic"

Add our keys to the relay config:
rly chains edit umee-betanet-1 key wallet_name
rly chains edit kichain-t-4 key wallet_name
  
Change the timeout in config to 30 sec:
nano ~/.relayer/config/config.yaml
timeout: 30s  

Check our coins:
rly q balance umee-betanet-1
rly q balance kichain-t-4

If we have the coins, then initialize the light client in both networks with the command:
rly light init umee-betanet-1
rly light init kichain-t-4 -f

Try to generate a channel between the networks with the command:
rly paths generate umee-betanet-1 kichain-t-4 transfer --port=transfer

Output:
#Generated path(transfer), run 'rly paths show transfer --yaml' to see details

We will now open a channel for relay:
rly tx link transfer --debug

We got a error. Now open our config and delete all strings:
client-id:
connection-id:
channel-id:

Re-initialize the light client with the commands:
rly light init umee-betanet-1 -f
rly light init kichain-t-4 -f

Run the command to open the channel again:
rly tx link transfer --debug

My output:
# Channel created: [umee-betanet-1]chan{channel-0}port{transfer} -> [kichain-t-4]chan{channel-61}port{transfer}

Chek our channel. Use command:
rly paths list -d
# 0: transfer             -> chns(✔) clnts(✔) conn(✔) chan(✔) (umee-betanet-1:transfer<>kichain-t-4:transfer)

See your accounts, if you have not written:
rly keys list umee-betanet-1
rly keys list kichain-t-4

Lets do test transactions
Example:
rly transact transfer [src-chain-id] [dst-chain-id] [amount] [dst-addr] [flags] 

rly tx transfer kichain-t-4 umee-betanet-1 1000utki umee18vf00ffhg0uvttzql7eyt05xmy9z380ss00vrn

https://ki.thecodes.dev/tx/
98008FDCBCBB15E62CA99D9A8DB6B0F9805822445FF546ABF1FDB3C5CBBE444F
2E52D6BCBA60C62E637CC8611C4A55B0050BDB14559DF81D4FFB7AB3F1992258
C8B14C34CFE41E0981837C3DEEF5935958375C5211E002791EE2A8DC99437313

rly tx transfer umee-betanet-1 kichain-t-4 1000uumee tki1yntfxkft599rrq2tldn3x4mg5wtj3jemjad5cu

8EC2237FB07E197BA7334E68932BC45B4073030CAA0DEC57F3C6E947A457C3CC
999825266C5F7CD1E5FA7C1520DAB450AA600535E825251CC17A345B5253840D
243665EB9EA8F3CFAF9760174FBC28CBC1C7649B8D6DA0C29F1C759303622CFB

Let's configure the Relay service and send our payments remotely.

Make service:

sudo tee /etc/systemd/system/rlyd.service > /dev/null <<EOF
[Unit]
Description=relayer client
After=network-online.target, starsd.service
[Service]
User=$USER
ExecStart=$(which rly) start transfer
Restart=always
RestartSec=3
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable rlyd
sudo systemctl start rlyd

Look at the channel numbers in the configuration:
rly paths show transfer  -- yaml

Send test transactions:
kid tx ibc-transfer transfer transfer channel-61 umee18vf00ffhg0uvttzql7eyt05xmy9z380ss00vrn 1000utki --from jonrow_wallet --fees=5000utki --gas=auto --chain-id kichain-t-4 --home $HOME/kichain/kid
C464AA2B737C993255E4A22F2D98AA7DC45F8947D649A984414F747D878C2C0F
23D309B40A4191DD2B25BBBBD80C0CF9A0755187B4FA03357147F159F21B3019
umeed tx ibc-transfer transfer transfer channel-0 tki1yntfxkft599rrq2tldn3x4mg5wtj3jemjad5cu 1000uatolo --from hlopeople_wallet --fees=5000uumee --gas=auto --chain-id umee-betanet-1
84AEFB621027A86516F2B4842CCCBDCD3DE06D6F8DAE46FFA3622F97BD22E120
5B0D40680717F483E145E9CD82C933FE763B416AF1179F4DC0978165FFECCFCB

That's all, gentlemens!
