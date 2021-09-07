<p>We are looking at how to set up a repeater between Umee and Ki-chain networks.<br />The repeater is installed to server with Ki-chain node and we will remotely connect to Umee.</p>
<p><strong>Make sure that Umee support IBC</strong><br />umeed q ibc-transfer params<br />#receive_enabled: true<br />#send_enabled: true<br />if "true" then lets go!</p>
<h2>Install and configure Relay on the Ki-chain host</h2>
<p><strong>Install the repeater</strong></p>
<p>git clone https://github.com/cosmos/relayer.git<br />cd relayer<br />make install<br />cd</p>
<p><strong>Initializing the repeater</strong><br />rly config init</p>
<p><strong>Create a folder with network configuration and go to it:</strong><br />mkdir ~/rly_config<br />cd ~/rly_config</p>
<p><strong>Create a settings file for both networks:</strong></p>
<p>nano kichain-t4-config.json<br />{<br />"chain-id": "kichain-t-4",<br />"rpc-addr": "http://127.0.0.1:26657",<br />"account-prefix": "tki",<br />"gas-adjustment": 1.5,<br />"gas-prices": "0.025utki",<br />"trusting-period": "48h"<br />}</p>
<p>nano umee-betanet-1-config.json</p>
<p>{<br />"chain-id": "umee-betanet-1",<br />"rpc-addr": "http://global_ip_adress:26657",<br />"account-prefix": "umee",<br />"gas-adjustment": 1.5,<br />"gas-prices": "0.025uumee",<br />"trusting-period": "48h"<br />}</p>
<p><strong>Now parse these settings into the config of the relay</strong><br />rly chains add -f umee-betanet-1-config.json<br />rly chains add -f kichain-t4-config.json<br />cd</p>
<p><br /><strong>We need to create new wallets:</strong><br />rly keys add umee-betanet-1 wallet_name<br />rly keys add kichain-t-4 wallet_name</p>
<p><strong>So you can restore your wallets, use commands:</strong><br />rly keys restore umee-betanet-1 wallet_name "mnemonic"<br />rly keys restore kichain-t-4 wallet_name "mnemonic"</p>
<p><strong>Add our keys to the relay config:</strong><br />rly chains edit umee-betanet-1 key wallet_name<br />rly chains edit kichain-t-4 key wallet_name<br /><br /><strong>Change the timeout in config to 30 sec:</strong><br />nano ~/.relayer/config/config.yaml<br />timeout: 30s</p>
<p><strong>Check our coins:</strong><br />rly q balance umee-betanet-1<br />rly q balance kichain-t-4</p>
<p><strong>If we have the coins, then initialize the light client in both networks with the command:</strong><br />rly light init umee-betanet-1<br />rly light init kichain-t-4 -f</p>
<p><strong>Try to generate a channel between the networks with the command:</strong><br />rly paths generate umee-betanet-1 kichain-t-4 transfer --port=transfer</p>
<p>my output:<br />#Generated path(transfer), run 'rly paths show transfer --yaml' to see details</p>
<p><strong>We will now open a channel for relay:</strong><br />rly tx link transfer --debug</p>
<p><strong>Error, ok.. Now open our config and delete all strings:</strong><br />client-id:<br />connection-id:<br />channel-id:</p>
<p><strong>Re-initialize the light client with the commands:</strong><br />rly light init umee-betanet-1 -f<br />rly light init kichain-t-4 -f</p>
<p><strong>Run the command to open the channel again:</strong><br />rly tx link transfer --debug# Channel created: [umee-betanet-1]chan{channel-0}port{transfer} -&gt; [kichain-t-4]chan{channel-61}port{transfer}</p>
<p><strong>Chek our channel. Use command:</strong><br />rly paths list -d<br /># 0: transfer -&gt; chns(✔) clnts(✔) conn(✔) chan(✔) (umee-betanet-1:transfer&lt;&gt;kichain-t-4:transfer)</p>
<p><strong>See your accounts, if you have not written:</strong><br />rly keys list umee-betanet-1<br />rly keys list kichain-t-4</p>
<p><strong>Make test transactions:</strong><br />Example:<br />rly tx transfer [src-chain-id] [dst-chain-id] [amount] [dst-addr] [flags]</p>
<p>rly tx transfer kichain-t-4 umee-betanet-1 1000utki umee18vf00ffhg0uvttzql7eyt05xmy9z380ss00vrn</p>
<p>https://ki.thecodes.dev/tx/<br />98008FDCBCBB15E62CA99D9A8DB6B0F9805822445FF546ABF1FDB3C5CBBE444F<br />2E52D6BCBA60C62E637CC8611C4A55B0050BDB14559DF81D4FFB7AB3F1992258<br />C8B14C34CFE41E0981837C3DEEF5935958375C5211E002791EE2A8DC99437313</p>
<p>rly tx transfer umee-betanet-1 kichain-t-4 1000uumee tki1yntfxkft599rrq2tldn3x4mg5wtj3jemjad5cu</p>
<p>https://explorer-umee.nodes.guru/transactions/</p>
<p>8EC2237FB07E197BA7334E68932BC45B4073030CAA0DEC57F3C6E947A457C3CC<br />999825266C5F7CD1E5FA7C1520DAB450AA600535E825251CC17A345B5253840D<br />243665EB9EA8F3CFAF9760174FBC28CBC1C7649B8D6DA0C29F1C759303622CFB</p>
<p>Setup the Relay service and send our payments remotely.</p>
<p><strong>Make service:</strong></p>
<p>sudo tee /etc/systemd/system/rlyd.service &gt; /dev/null &lt;&lt;EOF<br />[Unit]<br />Description=relayer client<br />After=network-online.target, starsd.service<br />[Service]<br />User=$USER<br />ExecStart=$(which rly) start transfer<br />Restart=always<br />RestartSec=3<br />LimitNOFILE=65535<br />[Install]<br />WantedBy=multi-user.target<br />EOF</p>
<p>sudo systemctl daemon-reload<br />sudo systemctl enable rlyd<br />sudo systemctl start rlyd</p>
<p><strong>Look at the channel numbers in the configuration:</strong><br />rly paths show transfer  -- yaml</p>
<p><strong>Send test transactions:</strong><br />kid tx ibc-transfer transfer transfer channel-61 umee18vf00ffhg0uvttzql7eyt05xmy9z380ss00vrn 1000utki --from jonrow_wallet --fees=5000utki --gas=auto --chain-id kichain-t-4 --home $HOME/kichain/kid<br />C464AA2B737C993255E4A22F2D98AA7DC45F8947D649A984414F747D878C2C0F<br />23D309B40A4191DD2B25BBBBD80C0CF9A0755187B4FA03357147F159F21B3019<br />umeed tx ibc-transfer transfer transfer channel-0 tki1yntfxkft599rrq2tldn3x4mg5wtj3jemjad5cu 1000uatolo --from hlopeople_wallet --fees=5000uumee --gas=auto --chain-id umee-betanet-1<br />84AEFB621027A86516F2B4842CCCBDCD3DE06D6F8DAE46FFA3622F97BD22E120<br />5B0D40680717F483E145E9CD82C933FE763B416AF1179F4DC0978165FFECCFCB</p>
<p><strong>Done.</strong></p>
