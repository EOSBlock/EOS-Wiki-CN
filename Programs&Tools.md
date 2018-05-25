# Programs&Tools

## Programs

### nodeos

核心的EOSIO守护进程，可以使用插件配置以运行节点。示例中用于生产区块，专用API端点以及本地开发。

### cleos

cleos 是一个命令行工具，他与nodeos公开的REST API进行交互。要使用cleos，需要将IP地址和端口号添加到nodeos实例，并配置cleos加载eosio :: chain_api_plugin，下面是有关cleos所有命令的列表：

```sh
$ cleos
ERROR: RequiredError: Subcommand required
Command Line Interface to EOSIO Client
Usage: ./cleos [OPTIONS] SUBCOMMAND

Options:
  -h,--help                   Print this help message and exit
  -H,--host TEXT=localhost    the host where nodeos is running
  -p,--port UINT=8888         the port where nodeos is running
  --wallet-host TEXT=localhost
                              the host where keosd is running
  --wallet-port UINT=8888     the port where keosd is running
  -v,--verbose                output verbose messages on error

Subcommands:
  version                     Retrieve version information
  create                      Create various items, on and off the blockchain
  get                         Retrieve various items and information from the blockchain
  set                         Set or update blockchain state
  transfer                    Transfer token from account to account
  net                         Interact with local p2p network connections
  wallet                      Interact with local wallet
  benchmark                   Configure and execute benchmarks
  push                        Push arbitrary transactions to the blockchain
```

想要获得特定的subcommand的帮助，通过无参数运行即可：

```sh
$ cleos create
ERROR: RequiredError: Subcommand required
Create various items, on and off the blockchain
Usage: ./cleos create SUBCOMMAND

Subcommands:
  key                         Create a new keypair and print the public and private keys
  account                     Create a new account on the blockchain
  producer                    Create a new producer on the blockchain

$ cleos create account
ERROR: RequiredError: creator
Create a new account on the blockchain
Usage: ./cleos create account [OPTIONS] creator name OwnerKey ActiveKey

Positionals:
  creator TEXT                The name of the account creating the new account
  name TEXT                   The name of the new account
  OwnerKey TEXT               The owner public key for the account
  ActiveKey TEXT              The active public key for the account

Options:
  -s,--skip-signature         Specify that unlocked wallet keys should not be used to sign transaction
  -x,--expiration             set the time in seconds before a transaction expires, defaults to 30s
  -f,--force-unique           force the transaction to be unique. this will consume extra bandwidth and remove any protections against accidently issuing the same transaction multiple times
```

### keosd

加载钱包相关插件的EOSIO钱包守护程序，例如HTTP接口和RPC API。

### launcher

启动器应用程序简化了多个nodeos节点在局域网或网络中的分布。它可以通过CLI进行配置以组成每个节点的配置文件，在对等主机之间安全地分发这些文件，然后启动多个nodeos实例。

## Tools

### eosiocpp

通过eosiocpp来生成ABI规范文件。

 eosiocpp可以通过检查合同源代码中声明的类型的内容来生成ABI规范文件。

@abi注释必须用在类型声明所附的注释中，才能表明一个类型必须被导出到ABI（作为一个动作或一个表）。

注释的语法格式如下：

- @abi action [name name2 ... nameN]
- @abi table [index_type name] To generate the ABI file, **eosiocpp** must be called with the **-g** option.

```
➜ eosiocpp -g abi.json types.hpp
Generated abi.json ...
```

eosiocpp也可用于生成序列化/反序列化ABI规范中定义的类型的帮助函数。

```
➜ eosiocpp -g abi.json -gs types.hpp
Generated abi.json ...
Generated types.gen.hpp ...
```

### Examples

#### Declaring an action

```c++
#include <eoslib/types.hpp>
#include <eoslib/string.hpp>

//@abi action
struct action_name {
  uint64_t    param1;
  uint64_t    param2;
  eosio::string param3;
};
{
  "types": [],
  "structs": [{
      "name": "action_name",
      "base": "",
      "fields": {
        "param1": "uint64",
        "param2": "uint64",
        "param3": "string"
      }
    }
  ],
  "actions": [{
      "action_name": "actionname",
      "type": "action_name"
    }
  ],
  "tables": []
}
```

### Declaring multiple actions using the same interface.

```c++
#include <eoslib/types.hpp>
#include <eoslib/string.hpp>

//@abi action action1 action2
struct action_name {
  uint64_t param1;
  uint64_t param2;
  eosio::string   param3;
};
{
  "types": [],
  "structs": [{
      "name": "action_name",
      "base": "",
      "fields": {
        "param1": "uint64",
        "param2": "uint64",
        "param3": "string"
      }
    }
  ],
  "actions": [{
      "action_name": "action1",
      "type": "action_name"
    },{
      "action_name": "action2",
      "type": "action_name"
    }
  ],
  "tables": []
}
```

### Declaring a table

```c++
#include <eoslib/types.hpp>
#include <eoslib/string.hpp>

//@abi table
struct my_table {
  uint64_t key;
};
{
  "types": [],
  "structs": [{
      "name": "my_table",
      "base": "",
      "fields": {
        "key": "uint64"
      }
    }
  ],
  "actions": [],
  "tables": [{
      "table_name": "mytable",
      "index_type": "i64",
      "key_names": [
        "key"
      ],
      "key_types": [
        "uint64"
      ],
      "type": "my_table"
    }
  ]
}
```

### Declaring a table with explicit index type

*a struct with 3 uint64_t can be both i64 or i64i64i64

```
#include <eoslib/types.hpp>

//@abi table i64
struct my_new_table {
  uint64_t key;
  uint64_t name;
  uint64_t age;
};
{
  "types": [],
  "structs": [{
      "name": "my_new_table",
      "base": "",
      "fields": {
        "key": "uint64",
        "name": "uint64",
        "age": "uint64"
      }
    }
  ],
  "actions": [],
  "tables": [{
      "table_name": "mynewtable",
      "index_type": "i64",
      "key_names": [
        "key"
      ],
      "key_types": [
        "uint64"
      ],
      "type": "my_new_table"
    }
  ]
}
```

#### Declaring a table and an action that use the same struct.

```
#include <eoslib/types.hpp>
#include <eoslib/string.hpp>

/*
 * @abi table
 * @abi action
 */ 
struct my_type {
  eosio::string key;
  eosio::name value;
};
{
  "types": [],
  "structs": [{
      "name": "my_type",
      "base": "",
      "fields": {
        "key": "string",
        "value": "name"
      }
    }
  ],
  "actions": [{
      "action_name": "mytype",
      "type": "my_type"
    }
  ],
  "tables": [{
      "table_name": "mytype",
      "index_type": "str",
      "key_names": [
        "key"
      ],
      "key_types": [
        "string"
      ],
      "type": "my_type"
    }
  ]
}
```

### Example of typedef exporting

```
#include <eoslib/types.hpp>
struct simple {
  uint64_t u64;
};

typedef simple simple_alias;
typedef eosio::name name_alias;

//@abi action
struct action_one : simple_alias {
  uint32_t u32;
  name_alias name;
};
{
  "types": [{
      "new_type_name": "simple_alias",
      "type": "simple"
    },{
      "new_type_name": "name_alias",
      "type": "name"
    }
  ],
  "structs": [{
      "name": "simple",
      "base": "",
      "fields": {
        "u64": "uint64"
      }
    },{
      "name": "action_one",
      "base": "simple_alias",
      "fields": {
        "u32": "uint32",
        "name": "name_alias"
      }
    }
  ],
  "actions": [{
      "action_name": "actionone",
      "type": "action_one"
    }
  ],
  "tables": []
}
```

#### Using the generated serialization/deserialization functions

```
#include <eoslib/types.hpp>
#include <eoslib/string.hpp>

struct simple {
  uint32_t u32;
  fixed_string16 s16;
};

struct my_complex_type {
  uint64_t u64;
  eosio::string str;
  simple simple;
  bytes bytes;
  public_key pub;
};

typedef my_complex_type complex;

//@abi action
struct test_action {
  uint32_t u32;
  complex cplx;
};
void apply( uint64_t code, uint64_t action ) {
   if( code == N(mycontract) ) {
      if( action == N(testaction) ) {
        auto msg = eosio::current_message<test_action>();
        eosio::print("test_action content\n");
        eosio::dump(msg);
         
        bytes b = eosio::raw::pack(msg);
        printhex(b.data, b.len);
     }
  }
}
```

### NOTE: table names and action names cannot use an underscore ("_").

### Calling contract with test values

```
cleos push message mycontract testaction '{"u32":"1000", "cplx":{"u64":"472", "str":"hello", "bytes":"B0CA", "pub":"EOS8CY2pCW5THmzvPTgEh5WLEAxgpVFXaPogPvgvVpVWCYMRdzmwx", "simple":{"u32":"164","s16":"small-string"}}}' -S mycontract
```

#### Will produce the following output in nodeos console

```
test_action content
u32:[1000]
cplx:[
  u64:[472]
  str:[hello]
  simple:[
    u32:[164]
    s16:[small-string]
  ]
  bytes:[b0ca]
  pub:[03b41078f445628882fe8c1e629909cbbd67ff4b592b832264dac187ac730177f1]
]
e8030000d8010000000000000568656c6c6fa40000000c736d616c6c2d737472696e6702b0ca03b41078f445628882fe8c1e629909cbbd67ff4b592b832264dac187ac730177f1
```

 