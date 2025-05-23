- [Mining Guidance](#mining-guidance)
  - [Instructions](#instructions)
  - [SGX](#sgx)
  - [Running the Service](#running-the-service)
    - [Preparing an Account](#preparing-an-account)
      - [Option 1](#option-1)
      - [Option 2](#option-2)
    - [Preparing Coin](#preparing-coin)
    - [Configuration Modification](#configuration-modification)
    - [Startup and Maintenance](#startup-and-maintenance)
      - [Update Device](#update-device)
      - [Exiting the Service (if required)](#exiting-the-service-if-required)
  - [FAQ](#faq)

# Mining Guidance

## Instructions

Before starting, please cofirm on [Intel© Ark](https://ark.intel.com/content/www/us/en/ark.html#@Processors) whether your processor is compatible with [Intel© SGX](https://www.intel.com/content/www/us/en/developer/tools/software-guard-extensions/overview.html).

Start by cloning the repository.

```bash
git clone https://github.com/deepsafe/def-mining-scripts.git
```

## SGX

Inspect your system's SGX support with:

```shell
./sgx-detect
```

Install sgx driver:
```
apt update
apt install  build-essential  automake autoconf libtool wget python libssl-dev dkms
wget https://download.01.org/intel-sgx/latest/linux-latest/distro/ubuntu18.04-server/sgx_linux_x64_driver_1.41.bin
bash sgx_linux_x64_driver_1.41.bin
```

Sample Output:

```text
✔  SGX instruction set
  ✔  CPU support
  ✔  CPU configuration
  ✔  Enclave attributes
  ✔  Enclave Page Cache
  SGX features
    ✔  SGX2  ✔  EXINFO  ✘  ENCLV  ✘  OVERSUB  ✔  KSS
    Total EPC size: 16.0GiB
✔  Flexible launch control
  ✔  CPU support
  ？ CPU configuration
  ✔  Able to launch production mode enclave
✔  SGX system software
  ✔  SGX kernel device (/dev/sgx_enclave)
  ✘  libsgx_enclave_common
  ✘  AESM service
  ✔  Able to launch enclaves
    ✔  Debug mode
    ✔  Production mode
    ✔  Production mode (Intel whitelisted)
```

If it displays as `✘ SGX kernel device (/dev/sgx_enclave)`, We should install SGX Environment and restart with:

```shell
sudo chmod +x sgx_enable
sudo ./sgx_enable
sudo reboot
```

## Running the Service

After confirming that your machine supports SGX2, you can proceed to launch the keyring service. The keyring service relies on obtaining events and state from a node service. In the configuration file, it is advisable to use an official node as the data source. Alternatively, you can initiate a local full node and utilize it as a data source once data synchronization is finished.

### Preparing an Account

Before initiating the process, you must create an account to serve as the owner responsible for holding and managing the current keyring service.

#### Option 1

Generate an account using the command `docker run -it --rm deepsafe/def-node:release identity generate`.

You will receive an output like this:

```text
Secret seed:      0x71235e1458ce9d140c8b8ded28ccc1e32e62c340aef51a65e1350a387dbe08a6
Public key (hex): 0x0248e7f02dcc9f7061a090b67dede93d7381847e94955aee7996603d2225e9f77e
Account ID:       0x34a5572cb21d34354e3091564d5edc7b791e9d5f
```

`Secret seed` signifies the account's private key, which can be directly imported into wallets such as MetaMask.
`Account ID` represents the account's address.

#### Option 2

An alternative approach is to create an account using MetaMask since DeepSafe's account system is Ethereum-compatible.

We recommend using MetaMask here because subsequent operations will require interaction with the [crva dashboard](https://crva.deepsafe.network/beta_mainnet).

### Preparing Coin

Prepare some DEF with your address to make sure for the deployment.

### Configuration Modification

For the majority of users, just substitute the `device_owner` in the default configuration file with the `Account ID` created in the previous step. There is no need to modify other parameters.

For example：
Open the `keyring.toml` file under the `configs` directory and replace `0x00000000000000000000000000000000000000`with your `<Account ID>`。

The default configuration file, encompassing identity information, service ports, P2P network, service launch types, etc., is as follows：

```toml
node_ws_url = "ws://127.0.0.1:9944"
# local node_call server port.
node_call_port = 8720
# the owner address of device （ETH type format）
device_owner = "0x00000000000000000000000000000000000000"
# database path
db_path = "/host/data"
# tokio console port
console_port = 5555

# database start option
[db_option]
create_if_missing = true
atomic_flush = true

[prime_factory_config]
threads = 5
target = 500

[network_config]
protocol_id = "betatestnet"
port = 38700
boot_nodes =["/ip4/172.210.130.200/tcp/38701/p2p/12D3KooWQBrkBWb3tLoUpxqXebxg1Eab24LfcFP3hv37ZF2c6qgz","/ip4/20.81.161.179/tcp/38701/p2p/12D3KooWMDqap7HMjA6nos1HpHpWt8JBcPepnZgYSd5PPmovAqD7"]
share_peer_interval = 30
is_mdns = true
is_autonat = true
only_global_ips = true
#max_peers_connected = 50
#peer_key = "0x0000000000000000000000000000000000000000000000000000000000011111"
#external_multiaddrs = ["/ip4/127.0.0.1/tcp/38700"]

[key_server_config]
attestation_style = 2 #This corresponds to using an image, epid=1, dcap=2
seal_policy = "MRENCLAVE"
exe_policy = { Multiply = { executors = 8 } }
round_time_limit = 180
clear_msg_interval = 360
```

Parameter Descriptions:

- **`node_ws_url`**: The accessible endpoint of the node service. If using a local port, it might be `ws://127.0.0.1:9944`.

- **`node_call_port`**: The port number through which the keyring service is exposed to the outside world.

- **`identity`**: The owner of the keyring service, a crucial factor affecting income and penalties for providing services.

- **`db_path`**: The storage path for the keyring service to persist data. It is not recommended to modify this. If you need to change it, please refer to the [occlum file system](https://occlum.readthedocs.io/en/latest/filesystem/fs_overview.html).

- **`db_option.create_if_missing`**: Runtime parameters for the RocksDB database exposed by the keyring service.

- **`db_option.atomic_flush`**: Runtime parameters for the RocksDB database exposed by the keyring service.
  
- **`prime_factory_config.threads`**: The number of threads that will be occupied when a new version is launched, with each version being initialized and called once, will occupy CPU for a period of time. In order to avoid occupying all CPU, adjustments can be made as appropriate (by default, all threads are occupied).
- **`prime_factory_config.target`**: The target number for generating safe prime numbers should be slightly larger during actual running, preferably between 100 and 1000. The larger the number, the longer the initialization time (default value is 500).

- **`network_config.protocol_id`**: The division of P2P network protocols is particularly important. Different networks have different `protocol_id`. Please follow the official configuration, otherwise the link will be invalid.

- **`network_config.port`**: The local port number for the keyring service's P2P.

- **`network_config.is_mdns`**: MDNS discovery enabled.

- **`network_config.is_autonat`**: Autonat discovery enabled.
  
- **`network_config.max_peers_connected`**: Maximum number of nodes allowed to be connected.

- **`network_config.boot_nodes`**: Information for the keyring service's P2P module to connect to other services. If configured incorrectly, it will become an isolated node and cannot participate in the service.

- **`network_config.share_peer_interval`**: The interval at which the keyring service's P2P module outputs the number of node connections.

- **`network_config.only_global_ips`**: Whether the keyring service's P2P module only manages public IP addresses.

- **`network_config.peer_key`**: Specifies the keyring service's P2P identity information. If not filled, it will be generated randomly.

- **`key_server_config.attestation_style`**: The mode of SGX remote attestation for the keyring service, where `1` represents `EPID` and `2` represents `DCAP`.

- **`key_server_config.seal_policy`**: The data encryption method for the keyring service, supporting `MRSIGNER` and `MRENCLAVE`. It has the same meaning as [Intel SGX sealing](https://www.intel.com/content/www/us/en/developer/articles/technical/introduction-to-intel-sgx-sealing.html). `MRSIGNER` trusts the software publisher, and the advantage is that it is compatible with historical data after software upgrades. `MRENCLAVE` only trusts the code, and the disadvantage is that it cannot read historical data after software upgrades.

- **`key_server_config.exe_policy`**: Optional execution engine that affects software execution efficiency. Generally, it does not need to be changed.

- **`key_server_config.round_time_limit`**: The waiting time in seconds for data interaction between keyring services. The session ends if it exceeds the waiting time.

- **`key_server_config.clear_msg_interval`**: The interval in seconds for the keyring service to clear abnormal data.


We employ Docker Compose for service management. If you need to specify a storage directory, you can modify the disk mapping in the `docker-compose.yaml` file to `./data`. By default, the data for the keyring service is stored in the same directory as the `docker-compose.yaml` file.


```text
volumes:
    - ./configs:/configs
    - ./data:/root/occlum_instance/data
```

Note: `/root/occlum_instance/data`  is an internal directory within Occlum and does not require modification.

### Startup and Maintenance

Before starting, we should check if `docker compose` is installed on the system. You can check this by running `docker compose --version ` or `docker-compose --version`. If it's not installed, you'll need to install it.

```shell
# install docker-compose
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
docker-compose --version
```

To start and view logs, use the following commands:

```shell
docker-compose up -d
docker-compose logs -f 
```

Wait for the software to run. In case of any errors, consult the [FAQ](#FAQ).

If the software is running correctly, you will observe logs similar to the following in the terminal:

```text
register sgx: "0x13bec2ac21b038d885d49d8100d307ce7761cf890bbdf25962a0eb2f2ac18101"
```

Login your `device_owner` account to [DeepSafe's CRVA](https://crva.deepsafe.network/beta_mainnet), unlisted devices will initially appear in the device list:

![crva-unlist](./images/crva-unlist.png)

**All subsequent actions will require Metamask signature. Please verify that the connected account in Metamask matches the `device_owner` account in your `keyring.toml` file to ensure consistency.**

#### Update Device

Go to the [DeepSafe's CRVA](https://crva.deepsafe.network/beta_mainnet) to activate the device. You need to vote tokens for the first time.

![crva-launch](./images/crva-launch.png)

For quick start, we need to stake 20000tBol at a time, and then click the `Submit` button.

![crva-submit](./images/crva-submit.png)

Wait for a epoch, and after the total stake amount reaches the condition (20000tBol), participate in the service through the 'Join Service'.

![crva-join](./images/crva-join.png)

When you see the device status change to `Service`, **congratulations** - the process is complete.

![crva-joined](./images/crva-joined.png)

> Check if the software is running correctly, indicated by the following logs: 
> HeartBeat session: 40167, challenge: [124, 148, 169, 145, 235, 214, 178, 134, 90, 10, 228, 25, 131, 65, 254, 0, 98, 93, 83, 204, 48, 182, 48, 209, 19, 158, 45, 233, 49, 254, 25, 129], hash: "0xa746ff7daae0952967cc9eadb38e6627052cd073cf0a319cb8fcb65e0abdabef"

#### Exiting the Service (if required)

Note: The system penalizes malicious nodes by deducting their staked tokens. To avoid financial losses due to irregular exits, please follow the process below to exit.

Exit the service by executing `Exit Service`:

![crva-exit](./images/crva-exit.png)

After executing `Exit Service`, you need to wait for a epoch before you can execute `Remove Device`. You can't perform any operations during this period.

Finally, stop your keyring service.

```shell
docker compose down
```

## FAQ

<span id="FAQ"> </span>

**If there is no device registration information on Boolscan or you receive the error message： `register failed for "Rpc error: RPC error: RPC call failed: ErrorObject { code: ServerError(1010), message: \"Invalid Transaction\", data: Some(RawValue(\"Custom error: 28\")) }`**

It indicates that keyring version number does not match.

**If you encounter an error during startup with the message: `[get_platform_quote_cert_data ../qe_logic.cpp:388] Error returned from the p_sgx_get_quote_config API. 0xe011.  Or [get_platform_quote_cert_data ../qe_logic.cpp:378] Error returned from the p_sgx_get_quote_config API. 0xe019`**

0xe011 means "The platform library doesn't have any platfrom cert data". If you set up the PCCS service by yourself, please follow [intel guide](https://www.intel.com/content/www/us/en/developer/articles/guide/intel-software-guard-extensions-data-center-attestation-primitives-quick-install-guide.html) strictly. If you run in cloud, Use the pccs service provided by the cloud service provider. 

```text
Azure "pccs_url": "https://global.acccache.azure.net/sgx/certification/v3"
Ali "pccs_url": "https://sgx-dcap-server.cn-hangzhou.aliyuncs.com/sgx/certification/v3/"
```

**If you encounter an error during startup with the message: `[ERROR] occlum-pal:     SIGILL Caught ! (line 37, file src/pal_check_fsgsbase.c) [ERROR] occlum-pal: FSGSBASE enablement check failed. (line 89, file src/pal_api.c`**

```
git clone https://github.com/occlum/enable_rdfsbase.git
cd enable_rdfsbase 
make && make install
```
