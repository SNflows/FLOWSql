# FLOWSql - FLOWS SQL query commandline client

This repo hold the binary files for FLOWSql.

## Download
Go to [release](https://github.com/SNflows/FLOWSql/releases) page and download the binary for your OS and architecture.

> Apple silicon users download `mac-arm` binary.

## Setup
You can straight-away run the binary. 

If you want to make it availabe system-wide, run these in your terminal.

```bash
sudo mv path/to/flowsql-download-binary  /usr/local/bin/flowsql
sudo chmod 755 /usr/local/bin/flowsql
```

You will need api key for the FLOWS and ask the FLOWS admins to enable you as database user. Then set the `FLOWS_API_KEY=<key>` in your shell environment.

Run `flowsql -h` for help and more details on recommended way of setting `FLOWS_API_KEY` in you system environment.

