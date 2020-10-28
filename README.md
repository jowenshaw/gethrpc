# gethrpc
extract go-ethereum/rpc as a separate module

# Example

```golang
package main

import (
	"context"
	"math/big"
	"os"

	log "github.com/jowenshaw/gethlog"
	rpc "github.com/jowenshaw/gethrpc"
)

var mlog log.Logger

func initLogger(module string) {
	if module != "" {
		mlog = log.New("module", module)
	} else {
		mlog = log.New()
	}
	// the log has two output:
	// 1. to stderr with colored format
	// 2. to file with json format
	mlog.SetHandler(log.MultiHandler(
		log.StreamHandler(os.Stderr, log.TerminalFormat(true)),
		log.LvlFilterHandler(
			log.LvlInfo,
			log.Must.FileHandler("testrpc.log", log.JSONFormat()))))
}

func main() {
	initLogger("testrpc")
	if len(os.Args) < 2 {
		mlog.Crit("Usage: please provide a url param")
	}

	url := os.Args[1]

	// set up connection
	client, err := rpc.Dial(url)
	if err != nil {
		mlog.Crit("client connect failed", "url", url, "err", err)
	}
	mlog.Info("client connect success", "url", url)
	defer client.Close()

	// post RPC request
	var result string
	err = client.CallContext(context.Background(), &result, "eth_blockNumber")
	if err != nil {
		mlog.Error("call eth_blockNumber failed", "err", err)
		return
	}

	// process the response result
	blockNumber, ok := new(big.Int).SetString(result, 0)
	if !ok {
		mlog.Error("result of block number is invalid number", "result", result)
		return
	}
	mlog.Info("get latest block number success", "number", blockNumber)
}
```
