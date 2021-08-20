
# pyWalletConnect


### A WalletConnect implementation for wallets in Python

A Python3 library to link a wallet with a WalletConnect web3 app. This library connects a Python wallet with a web3 app online, using [the WalletConnect v1 standard](https://docs.walletconnect.org/v/1.0/).

Thanks to WalletConnect, a Dapp is able to send JSON-RPC call requests to be handled by the Wallet, sign requests for transactions or messages remotely. Using WalletConnect, the wallet is a JSON-RPC service that the dapp can query through an encrypted tunnel and an online relay. This library is built for the wallet part, which establishes a link with the dapp and receives requests form a web3 app.

pyWalletConnect manages automatically on its own all the WalletConnect stack :

```
WalletConnect
    |
 JSON-RPC
    |
EncryptedTunnel
    |
 WebSocket
    |
   HTTP
    |
   TLS
    |
  Socket
```

## Installation and requirements

Works with Python >= 3.6.

### Installation of this library

Easiest way :  
`python3 -m pip install pyWalletConnect`  

From sources, download and run in this directory :  
`python3 -m pip  install .`

### Use

Instanciate with `pywalletconnect.WCClient.from_wc_uri`, then use methods functions of this object.

Basic example :

```python
from pywalletconnect import WCClient
# Input the wc URI
string_uri = input("Input the WalletConnect URI : ")
wallet_dapp = WCClient.from_wc_uri(string_uri)
# Wait for the sessionRequest info
try:
    req_id, req_chain_id, request_info = wallet_dapp.open_session()
except WCClientInvalidOption as exc:
    # In case error in the wc URI provided
    wallet_dapp.close()
    raise InvalidOption(exc)
if req_chain_id != account.chainID:
    # Chain id mismatch
    wallet_dapp.close()
    raise InvalidOption("Chain ID from Dapp is not the same as the wallet.")
# Display to the user request details provided by the Dapp.
user_ok = input(f"WalletConnect link request from : {request_info['name']}. Approve? [y/N]")
if user_ok.lower() == "y":
    # User approved
    wallet_dapp.reply_session_request(req_id, account.chainID, account.address)
    # Now the session with the Dapp is opened
    <...>
else:
    # User rejected
    wallet_dapp.close()
    raise UserInteration("user rejected the dapp connection request.")
```

pyWalletConnect maintains a TLS WebSocket opened with the host relay. It builds an internal pool of received request messages from the dapp.

Once the session is opened, you can read the pending messages received from the Dapp from time to time. And then your wallet app can process them.

Use a deamon thread timer for example, to call the `get_message()` method in a short time frequency. 3-6 seconds is an acceptable delay. This can also performed in a blocking *for* loop with a sleeping time. Then process the Dapp queries for further user wallet actions.

Remember to keep track of the request id, as it is needed for `.reply(req_id, result)` ultimately when sending the processing result back to the dapp service. One way is to provide the id in argument in your processing methods. Also this can be done with global or shared parameters.

```python
def process_sendtransaction(call_id, tx):
    # Processing the RPC query eth_sendTransaction
    # Collect the user approval about the tx query
    < Accept (tx) ? >
    # if approved :
    # Build and sign the provided transaction
    <...>
    # Broadcast the tx
    # Provide the transaction id as result
    result = "0x..." # Tx id
    wallet_dapp.reply(call_id, result)

def watch_messages():
    # Watch for messages received.
    # For WalletConnect calls reading.
    # Read all the message requests received from the dapp.
    # Then dispatch to the wallet service handlers.
    # get_message gives (id, method, params) or (None, "", [])
    wc_message = wallet_dapp.get_message()
    # Loop in the waiting messages pool, until depleted
    while wc_message[0] is not None:
        # Read a WalletConnect call message available
        id_request = wc_message[0]
        method = wc_message[1]
        parameters = wc_message[2]
        if "wc_sessionUpdate" == method:
            if parameters[0].get("approved") is False:
                raise Exception("Disconnected by the Dapp.")
        # Dispatch query processing
        elif "eth_signTypedData" == method:
            process_signtypeddata(id_request, parameters[0])
        elif "eth_sendTransaction" == method:
            tx_to_sign = parameters[0]
            process_sendtransaction(id_request, tx_to_sign)
        elif "eth_sign" == method:
            process_signtransaction(parameters[0])
        <...>
        # Next loop
        wc_message = wallet_dapp.get_message()


# GUI timer repeated or threading deamon
# Will call watch_messages every 4 seconds
apptimer = Timer(4000)
# Call watch_messages when expires periodically
apptimer.notify = watch_messages
```

See also the [RPC methods in WalletConnect](https://docs.walletconnect.org/v/1.0/json-rpc-api-methods/ethereum) to know more about the expected result regarding a specific RPC call.

## Interface methods of WCClient

`WCClient.from_wc_uri( wc_uri_str )`  
Create a WalletConnect wallet client from a wc URI.  
wc_uri_str : the wc full EIP1328 URI provided by the Dapp.  
You need to call *open_session* immediately after to get the session request info.

`.close()`  
Close the underlying WebSocket connection.

`.get_relay_url()`  
Give the URL of the WebSocket relay bridge.

`.get_message()`  
Get a RPC call message from the internal waiting list. pyWalletConnect maintains an internal pool of received request messages from the dapp. And this get_message method pops out a message in a FIFO manner : the first method call provides the oldest (first) received message. It can be used like a pump : call *get_message()* until an empty response. Because it reads a message from the receiving bucket one by one.  
Return : (RPCid, method, params) or (None, "", []) when no data were received since the last call (or from the initial session connection).  
Non-blocking, so always returns immediately.

`.reply( req_id, result_str )`  
Send a RPC response to the webapp (through the relay).  
req_id is the JSON-RPC id of the corresponding query request, where the result belongs to. One must kept track this id from the get_message, up to this reply. So a reply result is given back with its associated call query id.  
result_str is the result field to provide in the RPC result response.

`.open_session()`  
Start a WalletConnect session : wait for the session call request message.  
Must be called right after a WCClient creation.  
Returns : (message RPCid, chain ID, peerMeta data object).  
Or throws WalletConnectClientException("sessionRequest timeout")
after 8 seconds and no sessionRequest received.

`reply_session_request( msg_id, chain_id, account_address )`  
Send the sessionRequest result, when user approved the connection session request in the wallet.  
msg_id is the RPC id of the sessionRequest call provided by open_session.


## License

Copyright (C) 2021  BitLogiK SAS

This program is free software: you can redistribute it and/or modify  
it under the terms of the GNU General Public License as published by  
the Free Software Foundation, version 3 of the License.

This program is distributed in the hope that it will be useful,  
but WITHOUT ANY WARRANTY; without even the implied warranty of  
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  
See the GNU General Public License for more details.


## Support

Open an issue in the Github repository for help about its use.