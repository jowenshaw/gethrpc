# gethrpc
extract go-ethereum/rpc as a separate module

# Example

[Client Example](#client-example)

[Server Example](#server-example)

[Server Example 2](#server-example-2)

## Client Example

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

// client example:
// connect to ETH full node and rpc request the latest block number
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

## Server Example

```golang
package main

import (
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

	// show all log info of gethrpc module for debuging
	log.Root().SetHandler(log.LvlFilterHandler(log.LvlTrace, log.StderrHandler))
}

// server example:
// build a test rpc server which provide two methods `testrpc_hello` and `testrpc_echo`
func main() {
	initLogger("testrpc")
	if len(os.Args) < 2 {
		mlog.Crit("Usage: please provide a list port param")
	}

	listenPort := os.Args[1]

	server := rpc.NewServer()
	service := new(testService)

	serverName := "testrpc"
	err := server.RegisterName(serverName, service)
	if err != nil {
		mlog.Crit("register server failed", "err", err)
	}
	mlog.Info("register server success", "name", serverName)
	defer server.Stop()

	endpoint := "127.0.0.1:" + listenPort

	_, _, err = rpc.StartHTTPEndpoint(endpoint, rpc.DefaultHTTPTimeouts, server)
	if err != nil {
		mlog.Crit("start http endpoint failed", "endpoint", endpoint, "err", err)
	}

	mlog.Info("start http endpoint success", "endpoint", endpoint)
	exit := make(chan struct{})
	<-exit
}

type testService struct{}

type echoArgs struct {
	S string
}

type echoResult struct {
	String string
	Int    int
	Args   *echoArgs
}

// Hello api
// --> curl -X POST -H "Content-Type:application/json" --data '{"jsonrpc":"2.0","id":1,"method":"testrpc_hello","params":[]}' http://127.0.0.1:9999
// <-- {"jsonrpc":"2.0","id":1,"result":"hello"}
func (s *testService) Hello() string {
	return "hello"
}

// Echo api
// --> curl -X POST -H "Content-Type:application/json" --data '{"jsonrpc":"2.0","id":1,"method":"testrpc_echo","params":["hello", 123, {"S":"SSS"}]}' http://127.0.0.1:9999
// <-- {"jsonrpc":"2.0","id":1,"result":{"String":"hello","Int":123,"Args":{"S":"SSS"}}}
func (s *testService) Echo(str string, i int, args *echoArgs) echoResult {
	return echoResult{str, i, args}
}
```

## Server Example 2
```golang
package main

import (
	"os"
	"strconv"

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

	// show all log info of gethrpc module for debuging
	log.Root().SetHandler(log.LvlFilterHandler(log.LvlTrace, log.StderrHandler))
}

// server example:
// build a test rpc server which provide two methods `testrpc_hello` and `testrpc_echo`
func main() {
	initLogger("testrpc")
	if len(os.Args) < 2 {
		mlog.Crit("Usage: please provide a list port param")
	}

	listenPort, err := strconv.ParseInt(os.Args[1], 10, 32)
	if err != nil {
		mlog.Crit("wrong listen port number", "port", os.Args[1], "err", err)
	}

	srv := rpc.NewHTTPServer(mlog, rpc.DefaultHTTPTimeouts)

	apis := []rpc.API{
		{
			Namespace: "testrpc",
			Version:   "0.1.0",
			Service:   new(testService),
			Public:    true,
		},
	}
	conf := rpc.HTTPConfig{
		Modules:            []string{},
		CorsAllowedOrigins: []string{"*"},
		Vhosts:             []string{},
	}
	err = srv.EnableRPC(apis, conf)
	if err != nil {
		mlog.Crit("enable rpc failed", "err", err)
	}

	err = srv.SetListenAddr("localhost", int(listenPort))
	if err != nil {
		mlog.Crit("listen failed", "err", err)
	}

	err = srv.Start()
	if err != nil {
		mlog.Crit("start failed", "err", err)
	}
	defer srv.Stop()

	mlog.Info("start server success", "endpoint", srv.ListenAddr())
	exit := make(chan struct{})
	<-exit
}

type testService struct{}

type echoArgs struct {
	S string
}

type echoResult struct {
	String string
	Int    int
	Args   *echoArgs
}

// Hello api
// --> curl -X POST -H "Content-Type:application/json" --data '{"jsonrpc":"2.0","id":1,"method":"testrpc_hello","params":[]}' http://127.0.0.1:9999
// <-- {"jsonrpc":"2.0","id":1,"result":"hello"}
func (s *testService) Hello() string {
	return "hello"
}

// Echo api
// --> curl -X POST -H "Content-Type:application/json" --data '{"jsonrpc":"2.0","id":1,"method":"testrpc_echo","params":["hello", 123, {"S":"SSS"}]}' http://127.0.0.1:9999
// <-- {"jsonrpc":"2.0","id":1,"result":{"String":"hello","Int":123,"Args":{"S":"SSS"}}}
func (s *testService) Echo(str string, i int, args *echoArgs) echoResult {
	return echoResult{str, i, args}
}
```
