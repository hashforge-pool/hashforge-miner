# HashForge high performance CUDA GPU miner

## The binary is available on the [Releases](https://github.com/hashforge-pool/hashforge-miner/releases) page.

### Basic usage

`./hashforge run-gpu [OPTIONS] --algo <ALGO> --pool <POOL> --worker <WORKER> --address <ADDRESS>`

Options:

```
  -a, --algo <ALGO>                  Algorithm to run
  -p, --pool <POOL>                  Pool URL (stratum+tcp://pool:port or stratum+ssl://pool:port)
  -w, --worker <WORKER>              Worker name
  -a, --address <ADDRESS>            Wallet address
  -g, --gpus [<GPUS>...]             (Optional) Select gpus to use by index. example: -g 0,2,3 default: all
  -i, --idle-command <IDLE_COMMAND>  (Optional) Idle command to run when no work is available (currently disabled)
  -h, --help                         Print help
```

### Nock

`./hashforge run-gpu [OPTIONS] --algo nock --pool stratum+ssl://mine.nockhash.xyz:3333 --worker <WORKER> --address <ADDRESS>`
