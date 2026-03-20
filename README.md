# FLOWSql - FLOWS SQL query command-line client

This repo holds the binary files for FLOWSql.

## Download
Go to [release](https://github.com/SNflows/FLOWSql/releases) page and download the binary for your OS and architecture.

> Apple silicon users download `flowsql-darwin-arm64` binary.

## Setup
You can run the binary straight away. 

If you want to make it available system-wide, run these in your terminal.

```bash
sudo mv path/to/flowsql-download-binary  /usr/local/bin/flowsql
sudo chmod 755 /usr/local/bin/flowsql
```

You will need the API key for FLOWS and ask the FLOWS admins to enable you as a database user. Then set the `FLOWS_API_KEY=<key>` in your shell environment.

Run `flowsql -h` for help and more details on the recommended way of setting `FLOWS_API_KEY` in your system environment.

