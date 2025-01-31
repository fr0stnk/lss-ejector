
# Infrastructure Setup for Liquid Solo Staking

This guide outlines the steps to set up the infrastructure required for our liquid solo staking solution. It covers installing system dependencies, setting up Python 3.10 via PyEnv, installing Go 1.23.5, cloning and building necessary repositories, and configuring Supervisor to manage services.

---

## Table of Contents

- [Infrastructure Setup for Liquid Solo Staking](#infrastructure-setup-for-liquid-solo-staking)
  - [Table of Contents](#table-of-contents)
  - [System Dependencies](#system-dependencies)
  - [Install PyEnv and Python 3.10](#install-pyenv-and-python-310)
  - [Install Go 1.23.5](#install-go-1235)
  - [Clone Repositories](#clone-repositories)
  - [Build and Install Components](#build-and-install-components)
    - [Build deposit-cli](#build-deposit-cli)
    - [Build Ejector](#build-ejector)
    - [Build Relay](#build-relay)
  - [Generate Validator Keys](#generate-validator-keys)
  - [Configure and Run Services](#configure-and-run-services)
    - [Configure Ejector](#configure-ejector)
    - [Configure Relay](#configure-relay)
    - [Import Private Key for Relay](#import-private-key-for-relay)
    - [Supervisor Config for Voter](#supervisor-config-for-voter)
  - [Funding \& Running Services](#funding--running-services)
  - [Notes](#notes)

---

## System Dependencies

Update and upgrade existing packages, then install the required dependencies:

    apt update && apt upgrade -y
    apt install -y git build-essential libssl-dev zlib1g-dev libbz2-dev \
      libreadline-dev libsqlite3-dev wget curl llvm libncurses5-dev \
      libncursesw5-dev xz-utils tk-dev libffi-dev liblzma-dev \
      software-properties-common supervisor

---

## Install PyEnv and Python 3.10

1. **Clone pyenv** into `~/.pyenv`:

        git clone https://github.com/pyenv/pyenv.git ~/.pyenv

2. **Configure environment variables** (add these lines to `~/.bashrc` or `~/.zshrc`):

        export PYENV_ROOT="$HOME/.pyenv"
        export PATH="$PYENV_ROOT/bin:$PATH"
        eval "$(pyenv init --path)"
        eval "$(pyenv init -)"

3. **Reload your shell**:

        source ~/.bashrc

4. **Install Python 3.10** using pyenv and set it globally:

        pyenv install 3.10
        pyenv global 3.10

---

## Install Go 1.23.5

Download, extract, and configure Go:

    wget https://go.dev/dl/go1.23.5.linux-amd64.tar.gz
    rm -rf /usr/local/go
    tar -C /usr/local -xzf go1.23.5.linux-amd64.tar.gz
    export PATH=$PATH:/usr/local/go/bin
    source ~/.bashrc

---

## Clone Repositories

    git clone https://github.com/stratisproject/staking-deposit-cli
    git clone https://github.com/stratisproject/lss-ejector
    git clone https://github.com/stratisproject/lss-relay

---

## Build and Install Components

### Build deposit-cli

    cd staking-deposit-cli
    make build_linux

### Build Ejector

    cd ../lss-ejector/
    make install

### Build Relay

    cd ../lss-relay/
    make install

---

## Generate Validator Keys

1. Navigate to the `deposit-cli` directory:

        cd staking-deposit-cli

2. Run the key generation script:

        ./dist/deposit new-mnemonic

You will be prompted for:

- **Number of validators** (e.g., `50`) 
- **Withdrawal recipient address** (`<lss_staking_contract_address>`)  
- **Network/chain name** (e.g., `stratis` or `auroria`)  
- **Password** to secure your validator keystore(s)

Keep a note of the mnemonic phrase and the password. **Store these securely.**

The script will generate keys in the `validator_keys` directory, along with JSON file `deposit_data`.

> **Next**: You can load these validator keys into your validator using the Stratis Launcher.

---

## Configure and Run Services

### Configure Ejector

Create an Ejector configuration under Supervisor by editing (or creating) `/etc/supervisor/conf.d/lss-ejector.conf`:

    vi /etc/supervisor/conf.d/lss-ejector.conf

Paste in the following snippet (update `<keystore_password>` to match the password used to create your validator keys):

    [program:lss-ejector]
    command=lss-ejector start
      --execution_endpoint=http://localhost:8545
      --consensus_endpoint=http://localhost:3500
      --keys_dir=/root/staking-deposit-cli/validator_keys
      --staking_address=<lss_staking_contract_address>
    environment=KEYSTORE_PASSWORD=<keystore_password>
    redirect_stderr=true
    stdout_logfile=/var/log/supervisor-lss-ejector.log
    stdout_logfile_maxbytes=500MB
    stdout_logfile_backups=10
    autostart=false
    startretries=10
    stopwaitsecs=60

### Configure Relay

1. Generate a new Ethereum account (using MetaMask, etc.) and **export the private key**.

2. Create a new config file for the relay at `/opt/lss-relay/config.toml`:

    `vi /opt/lss-relay/config.toml`

Use the snippet below, replacing `<your_account>` with your newly created address:

    account = "<your_account>"
    eth2EffectiveBalance = 20000      # ether
    gasLimit = "3000000"
    maxGasPrice = "60000000000"       # wei
    batchRequestBlocksNumber = 32

    [pinata]
    apikey = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySW5mb3JtYXRpb24iOnsiaWQiOiI2ZTAyNGJlMS1mODQ4LTRiYmEtYjRlZS1hZWVhMGQ2MjNjNDgiLCJlbWFpbCI6ImNvcnBvcmF0ZUBzdHJhdGlzcGxhdGZvcm0uY29tIiwiZW1haWxfdmVyaWZpZWQiOnRydWUsInBpbl9wb2xpY3kiOnsicmVnaW9ucyI6W3siZGVzaXJlZFJlcGxpY2F0aW9uQ291bnQiOjEsImlkIjoiRlJBMSJ9LHsiZGVzaXJlZFJlcGxpY2F0aW9uQ291bnQiOjEsImlkIjoiTllDMSJ9XSwidmVyc2lvbiI6MX0sIm1mYV9lbmFibGVkIjpmYWxzZSwic3RhdHVzIjoiQUNUSVZFIn0sImF1dGhlbnRpY2F0aW9uVHlwZSI6InNjb3BlZEtleSIsInNjb3BlZEtleUtleSI6IjdkZTUwMDUzZDQwMzZlYjk1YTljIiwic2NvcGVkS2V5U2VjcmV0IjoiNTE3YmU5MDQ3YWIzNWU3YmU2YjlmOGM5OWJhMjAzN2E3MmU2Yzg0MGM4ZWI4N2MxNmY4MGNlODk5MzUzY2JiNiIsImV4cCI6MTc2MzY0ODUyNX0.Qp0QvflAg1yWIDHVzUmKZrMW0AivN7QWJAV_t_27xv4"
    pinDays = 5

    [contracts]
    stakingAddress = "<lss_staking_contract_address>"

    [[endpoints]]
    eth1 = "https://rpc.stratisevm.com"
    eth2 = "https://rpc.stratisevm.com/prysm"

### Import Private Key for Relay

    lss-relay import-account --base-path /opt/lss-relay

Follow the prompts to import your private key.

### Supervisor Config for Voter

Create a Supervisor configuration at `/etc/supervisor/config.d/lss-relay.conf`:

    vi /etc/supervisor/config.d/lss-relay.conf

Paste:

    [program:lss-relay]
    command=lss-relay start --base-path /opt/lss-relay
    environment=KEYSTORE_PASSWORD="<your_keystore_password>"
    redirect_stderr=true
    stdout_logfile=/var/log/supervisor-lss-relay.log
    stdout_logfile_maxbytes=500MB
    stdout_logfile_backups=10
    autostart=true
    startretries=10
    stopwaitsecs=60

---

## Funding & Running Services

1. **Deposit STRAX**:  
   You should deposit **at least 10 STRAX** to the new account you created for relay.

2. **Update & Start Supervisor**:

       supervisorctl update
       supervisorctl start lss-relay
       supervisorctl start lss-ejector

---

## Notes

- **Validator Keys**: Keep your mnemonic phrase and keystore password safe.  
- **Ejector & Relay Logs**: Supervisor writes logs to `/var/log/supervisor-*.log`. Check these with the below commands:
>     tail -f /var/log/supervisor-lss-ejector.log
>     tail -f /var/log/supervisor-lss-ejector.log
