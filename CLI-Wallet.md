# CLI Wallet

## Overview

keosd程序位于eos/build/programs/keosd目录下，用作保存private keys，private keys用来签署要push到blockchain伤的transaction。keosd在你的本地机器上运行并且在本地存储你的private keys。

## How to Run keosd

通过以下命令启动keosd：

```
$ keosd 
```

默认情况下，keosd会在本地 ~/ 目录下生成一个eosio-wallet文件夹，其中包含一个config.ini文件。可以使用--config-dir参数在命令行中指定配置文件的位置。配置文件包含传入http连接的http服务器端点以及用于跨源资源共享的其他参数。

默认情况下，keosd会在～/eosio-wallet目录下存储wallets。Wallet文件命名准讯约定格式：$<wallet-name> .wallet$。例如，默认wallet将存储在default.wallet文件内。随着其他wallet的创建，将为每个wallet创建类似的文件。命名为“foo”的wallet将具有foo.wallet的相应文件。可以使用--data-dir参数在命令行中指定钱包数据文件夹的位置。

## Command Reference

与keosd交互的命令行工具称为cleos，它位于eos/build/program/cleos目录下。它提供了以下的命令与keosd进行交互：

### Create

```sh
$ cleos wallet create ${options}
```

Options:

-n,--name TEXT=default The name of the new wallet（钱包名）

If you don’t provide an optional name it creates a default wallet（不提供钱包名的情况下使用default）

### Open

打开已经创建的wallet。您需要打开wallet才能对其进行操作。

```sh
cleos wallet open ${options}
```

Options:

-n,--name TEXT The name of the wallet to open（-n 钱包文件名）

### Lock

对一个wallet加锁。

```
$ cleos wallet lock ${options}
```

Options:

-n,--name TEXT The name of the wallet to lock（-n 钱包文件名）

 注意：当你的钱包管理器（keosd或nodeos）关闭时，你的wallet将被锁定。重新启动钱包管理器后，你需要解锁wallet。

### Unlock wallet

```
$ cleos wallet unlock ${options}
```

Options:

-n,--name TEXT The name of the wallet to unlock 

--password TEXT The password returned by wallet create 

### Import private key into wallet

```sh
$ cleos wallet import ${options} key
```

Positionals:

key TEXT Private key in WIF format to import

Options:

-n,--name TEXT The name of the wallet to import key into

### List

List opened wallets, * = unlocked

```
$ cleos wallet list
```

### Keys

List of private keys from all unlocked wallets in WIF format.

```
$ cleos wallet keys
```

## Managing Wallets with nodeos

可以使用nodeos代替keosd管理wallet。使用nodeos管理wallet并不是首选项，但是更方便开发和测试。不建议同时使用nodeos管理wallet，这样做不会破坏任何东西，但是可能会使你感觉很混乱。

要使用nodeos管理wallet，必须为nodeos配置eosio :: wallet_api_plugin。可以在启动nodeos时通过命令行参数执行此操作，可以通过在nodeos的配置文件中定义它执行此操作。你仍然会使用cleos作为你的界面来执行wallet管理任务，区别在于cleos会将其请求指向nodeos。

通过在启动nodeos时增加`--plugin eosio::wallet_api_plugin`参数，让nodeos管理wallet。

更改配置文件，在配置文件中增加以下命令：

```
plugin = eosio::wallet_api_plugin 
```

注意，如果你使用nodeos管理wallet，那么在nodeos关闭时，你的wallet会处于locked状态，你需要在重新启动nodeos后先解锁你的wallet。

cleos的全部命令请参照Cleos Command Refnrence。