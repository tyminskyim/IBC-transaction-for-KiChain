# IBC-transaction-for-KiChain
### Manual for Ibc-transfer from kichain to Cygnusx-Osmo-1 , but can use with any cosmos chain

#### Here I will describe the process of creating a relayer between two cosmos networks Kichain and osmosis.

## PART I. INSTALLATION OF THE NETWORK CLIENT

### 1. Download and install the network client

    git clone https://github.com/osmosis-labs/osmosis
    cd osmosis
    git checkout v1.0.1
    make install
    cd ~
	
### 2. Create a wallet

    chain-maind keys add WALLET_NAME

### 3. We look at the address of your wallet

    chain-maind keys show WALLET_NAME -a
	
## PART II. RELAYER START

### 1. Download the relayer and install it

    git clone https://github.com/cosmos/relayer.git
    cd relayer
    make install
    cd
	
### 2. Initializing the relayer

    rly config init

### 3. Let's create a folder in which we will put folders with data for configuring the relayer

    mkdir rly_config
    cd rly_config

### 4. We create configs for the networks used:

#### Ki-chain

    nano kichain-t-4_config.json

    {
	"chain-id": "kichain-t-4",
	"rpc-addr": "http://127.0.0.1:26657",
	"account-prefix": "tki",
	"gas-adjustment": 1.5,
	"gas-prices": "0.025utki",
	"trusting-period": "48h"
	}

save ctrl+x yes enter 

#### Osmosis

    nano cygnusx-osmo-1_config.json

    {
	"chain-id": "cygnusx-osmo-1",
	"rpc-addr": "http://54.166.148.90:26657",
	"account-prefix": "osmo",
	"gas-adjustment": 1.5,
	"gas-prices": "0.025uosmox",
	"trusting-period": "48h"
	}
	
### 5. Add pre-written settings to the config file of the relayer

    rly chains add -f kichain-t-4_config.json
    rly chains add -f cygnusx-osmo-1_config.json
    cd
	
### 6. We add wallets in both networks to the relayer (it is recommended to create separate wallets for this). Remember to save your seed phrases and addresses

    rly keys add kichain-t-4 kidrly
    rly keys add cygnusx-osmo-1 osmorly

### 7. Add the created wallets to the config of the relayer

    rly chains edit kichain-t-4 key kidrly
    rly chains edit cygnusx-osmo-1 key osmorly
	
### 8. Change the waiting timeout in the relayer settings so that it can accurately process transactions

    nano ~/.relayer/config/config.yaml

Looking for a line

    timeout: 10s
	
Replace with

    timeout: 3m
	
### 9. We replenish the purses of the relayer

    kid tx bank send WALLET_NAME_KICHAIN_WALLET_ADDRESS 10000000utki --home PATH_TO_YOUR_NODE_DIRECTORY --chain-id kichain-t-4
	
	osmosisd tx bank send WALLET_NAME_OSMO WALLET_ADDRESS_osmorly 10000000uosmox --node http://54.166.148.90:26657/ --chain-id cygnusx-osmo-1 --fees 5000uosmox
	
### 10. We check that the tokens have arrived
    rly q balance kichain-t-4
    rly q balance cygnusx-osmo-1
	
### 11. When the tokens arrive on the wallets of the relay, we initialize clients for both networks

    rly light init kichain-t-4 -f
    rly light init cygnusx-osmo-1 -f
	
### 12. We create a tunnel between the two networks.

    rly paths generate kichain-t-4 cygnusx-osmo-1 transfer --port=transfer
	
### 13. We check that the tunnel was created successfully.

    rly paths list -d
	
#### you should see the following output

    0: transfer             -> chns(✔) clnts(✔) conn(✔) chan(✔) 
	(kichain-t-4:transfer<>cygnusx-osmo-1:transfer)
	
### 14. create a service file

    sudo tee /etc/systemd/system/rlyd.service > /dev/null <<EOF
	[Unit]
	Description=relayer client
	After=network-online.target, kichaind.service
	[Service]
	User=$USER
	ExecStart=$(which rly) start transfer
	Restart=always
	RestartSec=3
	LimitNOFILE=65535
	[Install]
	WantedBy=multi-user.target
	EOF
	
### 15. We start the service file

    sudo systemctl daemon-reload
    sudo systemctl enable rlyd
    sudo systemctl start rlyd

To check the logs of the relayer, use:
    journalctl -u rlyd -f
	
### 16. To carry out an ibc transaction, you can use both the subcommands of the relay and, accordingly, the wallets that are added to it, as well as the subcommands of the clients of the networks and the wallets added to them. Examples:

#### relayer

    rly tx transfer kichain-t-4 cygnusx-osmo-1 1000000utki USER_ADDRESS_On_OSMO_NETWORK --path transfer
	
#### Client KICHAIN

    kid tx ibc-transfer transfer transfer CHANNEL_ID RECIPIENT_ADDRESS IN_OSMO_NET 1000000utki --from WALLET_NAME_KICHAIN --home PATH_TO_YOUR_NODE_DIRECTORY_chain-id kichain-t-4
	
#### To get the CHANNEL ID use the command

    rly paths show transfer --yaml
	
#### You need the value of the channel-id item of the src section

### 17. After the transaction, we check the balance of the recipient's wallet to make sure that the tokens have arrived

    osmosisd q bank balances RECEIVER_ADDRESS_IN_OSMO_NETWORK --node http://54.166.148.90:26657/
