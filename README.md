# FLOWSql - FLOWS SQL query command-line client

This repo holds the binary files for FLOWSql.


## Setup

### Automated download and setup

```bash
sudo wget $(curl -sL https://install-scripts.bose.dev/detect-platform.sh | sh -s -- SNflows/FLOWSql flowsql) -O /usr/local/bin/flowsql && sudo chmod +x /usr/local/bin/flowsql
```
This will install the program appropriate for your CPU architecture,  and make it system-wide available.

### Manual setup

One can simply download the binary and run it.

To download, go to the [release](https://github.com/SNflows/FLOWSql/releases) page and download the binary for your OS and architecture.

> Apple silicon users download `flowsql-darwin-arm64` binary.


If you want to make it available system-wide, run this in your terminal.


```bash
sudo mv path/to/flowsql-download-binary  /usr/local/bin/flowsql
sudo chmod 755 /usr/local/bin/flowsql
```

## Usage

Run `flowsql -h` for exhaustive list of options and detailed usage instructions.

## Set API key

You will need the API key for FLOWS (get it from https://flows.phys.au.dk/myaccount.php) and  **also** ask the FLOWS admins to enable you as a database user. Then set the `FLOWS_API_KEY=<key>` in your shell environment.

Run `flowsql -h` for help and more details on the recommended way of setting `FLOWS_API_KEY` in your system environment.

## Upgrading the binary

The FLOWSql binary is capable of self-updating if there is a newer version available, run

```bash
flowsql --upgrade
```

