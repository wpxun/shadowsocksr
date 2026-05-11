# ShadowsocksR (SSR)

## Project Overview
ShadowsocksR is a fast tunnel proxy designed to help users bypass firewalls. It is a robust Python implementation that supports multiple cryptographic methods, obfs (obfuscation) plugins, and multi-user configurations via API or JSON file management.

## Architecture
The project primarily consists of two main components:
- **Server (`ssserver`):** Runs on a remote machine to receive traffic from the client, decrypt it, and forward it to the destination. Entry point: `shadowsocks/server.py`.
- **Client (`sslocal`):** Runs locally on the user's machine to intercept traffic, encrypt it, and forward it to the remote server. Entry point: `shadowsocks/local.py`.

### Key Directories & Files
- `shadowsocks/`: The core Python package.
  - `crypto/`: Cryptography modules supporting various ciphers (AES, ChaCha20, RC4, etc.) via OpenSSL or libsodium.
  - `obfsplugin/`: Obfuscation plugins to disguise proxy traffic.
- `tests/`: Contains test scripts (e.g., `test_command.sh`, `.json` configuration files) for automated testing.
- `apiconfig.py` / `config.json` / `mudb.json`: Used for managing server configurations and multi-user setups.
- `setup.py`: Standard Python setup script for packaging and installation.
- `initcfg.sh`: A helper bash script to quickly generate default configuration files like `user-config.json`.

## Building and Running

### Setup
The project can be used directly from source or installed as a package.
To initialize basic configurations (creates `user-config.json`):
```bash
bash initcfg.sh
```

To install the package globally (which provides `sslocal` and `ssserver` console commands):
```bash
python setup.py install
```

### Running the Server
You can run the server directly via Python by specifying arguments:
```bash
cd shadowsocks
python server.py -p 443 -k password -m aes-128-cfb -O auth_aes128_md5 -o tls1.2_ticket_auth_compatible
```
Or, if you have generated a configuration file, simply run:
```bash
cd shadowsocks
python server.py
```
There are also utility scripts provided for background execution in the root directory:
- `./logrun.sh`: Runs the server in the background.
- `./stop.sh`: Stops the background server.
- `./tail.sh`: Monitors the server log.

### Running the Client
```bash
cd shadowsocks
python local.py -s <server_ip> -p <server_port> -l 1080 -k <password> -m <encryption_method>
```

### Testing
Tests are managed via bash scripts that execute Python modules with `coverage`. To run the command line assertions:
```bash
bash tests/test_command.sh
```

## Development Conventions
- **Language Compatibility:** The codebase is written to support both Python 2 (2.6, 2.7) and Python 3 (3.3, 3.4), as well as PyPy. It makes heavy use of `from __future__ import ...` statements for compatibility.
- **License:** Apache License 2.0.
- **Configuration Management:** When adding new options, ensure they are compatible with existing JSON configurations and `apiconfig.py` structures. Avoid hardcoding defaults in the source if they can be placed in configuration templates.
