# 井字棋游戏

### 目标

接下来的教程将会指导用户来构建一个简单的玩家对玩家的游戏合约。我们将使用井字棋游戏来演示这一点。这篇教程的最终结果可以在[这里](https://github.com/EOSIO/eos/tree/master/contracts/tic_tac_toe)找到。

### 假设
对于这个游戏，我们将会使用一个标准的3x3的井字棋方阵。玩家将会被分为两类角色: **主场**和**挑战者**。主场总是走第一步。每一对玩家**只能**最多同时玩2个游戏，其中第一个玩家成为主场并且在另一场中第二个玩家成为主场。


#### 方阵
不像是在传统的井字棋游戏中使用`o`和`x`，我们用`1`来表示主场，用`2`来表示挑战者的移动，`0`来表示一个空格。因此，我们可以使用一维数组来存储这个方针。因此：

|           | (0,0) | (1,0) | (2,0) |
| :-------: | :---: | :---: | :---: |
| **(0,0)** |   -   |   o   |   x   |
| **(0,1)** |   -   |   x   |   -   |
| **(0,2)** |   x   |   o   |   o   |

假设 x 是主场，那么上面这个方阵就等价于:  `[0, 2, 1, 0, 1, 0, 1, 2, 2]`

#### 动作
用户将会有以下动作来与这个合约进行交互:

- create: 创建一个新的游戏
- restart: 重新开始一个已经存在的游戏，主场和挑战者都可以执行这个操作
- close: 关闭一个已经存在的游戏，这将会释放用来存储这个游戏的存储空间，只允许主场执行这个操作
- move: 做一个移动动作

#### 合约账户
在下面的教程中，我们将使用一个叫做`tic.tac.toe`的账户来提交合约。为了防止`tic.tac.toe`这个账户名称已经存在了，你可以使用其他账号名称来替换代码中的`tic.tac.toe`。如果你还没有创建账号，请先创建账号。

```bash
$ cleos create account ${creator_name} ${contract_account_name} ${contract_pub_owner_key} ${contract_pub_active_key} --permission ${creator_name}@active
# e.g. $ cleos create account inita tic.tac.toe  EOS4toFS3YXEQCkuuw1aqDLrtHim86Gz9u3hBdcBw5KNPZcursVHq EOS7d9A3uLe6As66jzN8j44TXJUqJSK3bFjjEEqR4oTvNAB3iM9SA --permission inita@active
```
请确保你的钱包已经解锁并且创建者的私钥已经导入了钱包，否则上面的命令会失败。

### 开始!
我们将会创建三个文件:
- tic_tac_toe.hpp => 定义合约结构的头文件
- tic_tac_toe.cpp => 合约的主逻辑
- tic_tac_toe.abi => 用户与合约交互的接口
  注意: 在这个例子中我们使用用户名为N(tic.tac.toe)来作为这个合约的账户。如果你想使用一个不同的名称，请把`tic.tac.toe`替换成你自己的账户名。

### 定义结构
让我们从头文件开始并且定义这个合约的结构。打开**tic_tac_toe.hpp**并且用下面的这个模版开始:

```cpp
// Import necessary library
#include <eosiolib/eosio.hpp> // Generic eosio library, i.e. print, type, math, etc

using namespace eosio;
namespace tic_tac_toe {
   static const account_name games_account = N(games);
   static const account_name code_account = N(tic.tac.toe);
    // Your code here
}
```

#### 游戏表
对于这个合约，我们需要一个表格来存储一个游戏列表。让我们来定义它：
```cpp
...
namespace tic_tac_toe {
    ...
    typedef eosio::multi_index< games_account, game> games;
}
  
```
- 第一个模版参数定义了这个表的名称
- 第二个模版参数定义了它所要存储的结构(我们将会在下一节中定义它)

#### 游戏结构
让我们来定义这个游戏的结构。请确保在代码中这个结构定义在表定义之前。

```cpp
...
namespace tic_tac_toe {
   static const uint32_t board_len = 9;
   struct game {
      game() {}
      game(account_name challenger, account_name host):challenger(challenger), host(host), turn(host) {
         // Initialize board
         initialize_board();
      }
      account_name     challenger;
      account_name     host;
      account_name     turn; // = account name of host/ challenger
      account_name     winner = N(none); // = none/ draw/ account name of host/ challenger
      uint8_t          board[9]; //

      // Initialize board with empty cell
      void initialize_board() {
         for (uint8_t i = 0; i < board_len ; i++) {
            board[i] = 0;
         }
      }

      // Reset game
      void reset_game() {
         initialize_board();
         turn = host;
         winner = N(none);
      }

      auto primary_key() const { return challenger; }

      EOSLIB_SERIALIZE( game, (challenger)(host)(turn)(winner)(board) )
   };
}
```
`primary_key`方法对于上面的游戏表格定义是必要的，因为它是这个表知道如何查找这个表的key是哪个字段的。

#### 动作结构
##### 创建游戏
为了创建这个游戏，我们需要主场账户名和挑战者的账户名。`EOSLIB_SERIALIZE`这个宏定义提供了序列化和反序列化方法，因此动作可以在合约与`nodeos`系统之间来回传递。

```cpp
...
namespace tic_tac_toe {
   ...
   struct create {
      account_name   challenger;
      account_name   host;

      EOSLIB_SERIALIZE( create, (challenger)(host) )
   };
   ...
}
```
##### 重启游戏
为了重新开始这个游戏，我们需要主场账户名称和挑战者账户名称来唯一标识这个游戏。并且，我们需要指定谁来重新开始这个游戏，因此我们需要验证合约签名是被提供的。

```cpp
...
namespace tic_tac_toe {
   ...
   struct restart {
      account_name   challenger;
      account_name   host;
      account_name   by;

      EOSLIB_SERIALIZE( restart, (challenger)(host)(by) )
   };
   ...
}
```
##### 关闭游戏
为了关闭这个游戏，我们需要主场账户名和挑战者账户名来标识这个游戏。

```cpp
...
namespace tic_tac_toe {
   ...
   struct close {
      account_name   challenger;
      account_name   host;

      EOSLIB_SERIALIZE( close, (challenger)(host) )
   };
   ...
}
```
##### 移动
为了做出一个移动，我们需要主账户名和挑战者账户名来标识这个游戏。并且，我们需要指定谁做出了这次移动以及他做出的移动。

```cpp
...
namespace tic_tac_toe {
   ...
   struct movement {
      uint32_t    row;
      uint32_t    column;

      EOSLIB_SERIALIZE( movement, (row)(column) )
   };

   struct move {
      account_name   challenger;
      account_name   host;
      account_name   by; // the account who wants to make the move
      movement       mvt;

      EOSLIB_SERIALIZE( move, (challenger)(host)(by)(mvt) )
   };
   ...
}
```
你可以在 [这里](https://github.com/EOSIO/eos/blob/master/contracts/tic_tac_toe/tic_tac_toe.hpp)看到最终的 tic_tac_toe.hpp

### 主函数
让我们打开`tic_tac_toe.cpp` 并且填充这个模版。
```cpp
#include "tic_tac_toe.hpp"
using namespace eosio;
/**
*  The apply() method must have C calling convention so that the blockchain can lookup and
*  call these methods.
*/
extern "C" {

   using namespace tic_tac_toe;
   /// The apply method implements the dispatch of events to this contract
   void apply( uint64_t receiver, uint64_t code, uint64_t action ) {
      // Put your action handler here
   }

} // extern "C"
```

#### 动作处理器
我们想 `tic_tac_toe` 这个合约只与发送到 `tic.tac.toe` 这个账户的动作交互并且根据动作类型做出不同表现。让我们添加一个`impl`结构，这个结构根据'on'重载方法使用不同动作类型（对于这个例子来说，这可能看起来有些过分，但是您会看到这种模式可用于您可以扩展的其他合约，如`currency`这个合约）：

```cpp
using namespace eosio;
namespace tic_tac_toe {
struct impl {
   ...
   /// The apply method implements the dispatch of events to this contract
   void apply( uint64_t receiver, uint64_t code, uint64_t action ) {

      if (code == code_account) {
         if (action == N(create)) {
            impl::on(eosio::unpack_action_data<tic_tac_toe::create>());
         } else if (action == N(restart)) {
            impl::on(eosio::unpack_action_data<tic_tac_toe::restart>());
         } else if (action == N(close)) {
            impl::on(eosio::unpack_action_data<tic_tac_toe::close>());
         } else if (action == N(move)) {
            impl::on(eosio::unpack_action_data<tic_tac_toe::move>());
         }
      }
   }

};
}

...
extern "C" {

   using namespace tic_tac_toe;
   /// The apply method implements the dispatch of events to this contract
   void apply( uint64_t receiver, uint64_t code, uint64_t action ) {
      impl().apply(receiver, code, action);
   }

} // extern "C"
```

注意我们在将其传递到一个具体的处理器之前使用了 `unpack_action_data<T>()`，`unpack_action_data<T>()`是将合约接收到的动作转换为`struct T`。

为了使事情变得整洁，我们将封装动作处理程序在`struct impl`中：

```cpp
...
struct impl {
...
   /**
    * @param create - action to be applied
    */
   void on(const create& c) {
      // Put code for create action here
   }

   /**
    * @brief Apply restart action
    * @param restart - action to be applied
    */
   void on(const restart& r) {
      // Put code for restart action here
   }

   /**
    * @brief Apply close action
    * @param close - action to be applied
    */
   void on(const close& c) {
      // Put code for close action here
   }

   /**
    * @brief Apply move action
    * @param move - action to be applied
    */
   void on(const move& m) {
      // Put code for move action here
   }

   /// The apply method implements the dispatch of events to this contract
   void apply( uint64_t receiver, uint64_t code, uint64_t action ) {
...
```

#### 创建动作处理器
对于创建游戏动作处理器，我们想：

1. 确保该操作具有来自主场的签名
2. 确保挑战者和主场不是同一个玩家
3. 确保没有现有的游戏
4. 将新创建的游戏存储到数据库中

```cpp
struct impl {
   ...
   /**
    * @brief Apply create action
    * @param create - action to be applied
    */
   void on(const create& c) {
      require_auth(c.host);
      eosio_assert(c.challenger != c.host, "challenger shouldn't be the same as host");

      // Check if game already exists
      games existing_host_games(code_account, c.host);
      auto itr = existing_host_games.find( c.challenger );
      eosio_assert(itr == existing_host_games.end(), "game already exists");

      existing_host_games.emplace(c.host, [&]( auto& g ) {
         g.challenger = c.challenger;
         g.host = c.host;
         g.turn = c.host;
      });
   }
   ...
}

```

#### 重启动作处理器
对于重新启动操作处理程序，我们希望：

1. 确保该动作来自主场/挑战者签名
2. 确保游戏存在
3. 确保重新启动操作由主场/挑战者完成
4. 重置游戏
5. 将更新的游戏存储到数据库

```cpp
struct impl {
   ...
   /**
    * @brief Apply restart action
    * @param restart - action to be applied
    */
   void on(const restart& r) {
      require_auth(r.by);

      // Check if game exists
      games existing_host_games(code_account, r.host);
      auto itr = existing_host_games.find( r.challenger );
      eosio_assert(itr != existing_host_games.end(), "game doesn't exists");

      // Check if this game belongs to the action sender
      eosio_assert(r.by == itr->host || r.by == itr->challenger, "this is not your game!");

      // Reset game
      existing_host_games.modify(itr, itr->host, []( auto& g ) {
         g.reset_game();
      });
   }
   ...
}

```

#### 关闭动作处理器
对于关闭操作处理程序，我们希望：

1. 确保该操作具有来自主场的签名
2. 确保游戏存在
3. 从数据库中删除游戏

```cpp
struct impl {
   ...
   /**
    * @brief Apply close action
    * @param close - action to be applied
    */
   void on(const close& c) {
      require_auth(c.host);

      // Check if game exists
      games existing_host_games(code_account, c.host);
      auto itr = existing_host_games.find( c.challenger );
      eosio_assert(itr != existing_host_games.end(), "game doesn't exists");

      // Remove game
      existing_host_games.erase(itr);
   }
   ...
}

```

#### 移动动作处理器
对于移动操作处理程序，我们希望：

1. 确保该动作来自主场/挑战者签名
2. 确保游戏存在
3. 确保游戏尚未完成
4. 确保移动操作由主场/挑战者完成
5. 确保这是正确的用户轮次
6. 验证移动是否有效
7. 用新动作更新棋局
8. 将move_turn更改为其他玩家
9. 确定是否有赢家
10. 将更新的游戏存储到数据库

```cpp
struct impl {
   ...
   bool is_valid_movement(const movement& mvt, const game& game_for_movement) {
      // Put code here
   }

   account_name get_winner(const game& current_game) {
      // Put code here
   }
   ...
   /**
    * @brief Apply move action
    * @param move - action to be applied
    */
   void on(const move& m) {
      require_auth(m.by);

      // Check if game exists
      games existing_host_games(code_account, m.host);
      auto itr = existing_host_games.find( m.challenger );
      eosio_assert(itr != existing_host_games.end(), "game doesn't exists");

      // Check if this game hasn't ended yet
      eosio_assert(itr->winner == N(none), "the game has ended!");
      // Check if this game belongs to the action sender
      eosio_assert(m.by == itr->host || m.by == itr->challenger, "this is not your game!");
      // Check if this is the  action sender's turn
      eosio_assert(m.by == itr->turn, "it's not your turn yet!");


      // Check if user makes a valid movement
      eosio_assert(is_valid_movement(m.mvt, *itr), "not a valid movement!");

      // Fill the cell, 1 for host, 2 for challenger
      const auto cell_value = itr->turn == itr->host ? 1 : 2;
      const auto turn = itr->turn == itr->host ? itr->challenger : itr->host;
      existing_host_games.modify(itr, itr->host, [&]( auto& g ) {
         g.board[m.mvt.row * 3 + m.mvt.column] = cell_value;
         g.turn = turn;

         //check to see if we have a winner
         g.winner = get_winner(g);
      });
   }
   ...
}

```
#### 移动验证
有效的移动被定义为在空白单元格内完成的移动：
```cpp
struct impl {
   ...
   /**
    * @brief Check if cell is empty
    * @param cell - value of the cell (should be either 0, 1, or 2)
    * @return true if cell is empty
    */
   bool is_empty_cell(const uint8_t& cell) {
      return cell == 0;
   }

   /**
    * @brief Check for valid movement
    * @detail Movement is considered valid if it is inside the board and done on empty cell
    * @param movement - the movement made by the player
    * @param game - the game on which the movement is being made
    * @return true if movement is valid
    */
   bool is_valid_movement(const movement& mvt, const game& game_for_movement) {
      uint32_t movement_location = mvt.row * 3 + mvt.column;
      bool is_valid = movement_location < board_len && is_empty_cell(game_for_movement.board[movement_location]);
      return is_valid;
   }
   ...
}
```
#### 获取赢家
赢家被定义为第一位成功将他们的三个标记放置在水平，垂直或对角线上的玩家。
```cpp
struct impl {
   ...
   /**
    * @brief Get winner of the game
    * @detail Winner of the game is the first player who made three consecutive aligned movement
    * @param game - the game which we want to determine the winner of
    * @return winner of the game (can be either none/ draw/ account name of host/ account name of challenger)
    */
   account_name get_winner(const game& current_game) {
      if((current_game.board[0] == current_game.board[4] && current_game.board[4] == current_game.board[8]) ||
         (current_game.board[1] == current_game.board[4] && current_game.board[4] == current_game.board[7]) ||
         (current_game.board[2] == current_game.board[4] && current_game.board[4] == current_game.board[6]) ||
         (current_game.board[3] == current_game.board[4] && current_game.board[4] == current_game.board[5])) {
         //  - | - | x    x | - | -    - | - | -    - | x | -
         //  - | x | -    - | x | -    x | x | x    - | x | -
         //  x | - | -    - | - | x    - | - | -    - | x | -
         if (current_game.board[4] == 1) {
            return current_game.host;
         } else if (current_game.board[4] == 2) {
            return current_game.challenger;
         }
      } else if ((current_game.board[0] == current_game.board[1] && current_game.board[1] == current_game.board[2]) ||
                 (current_game.board[0] == current_game.board[3] && current_game.board[3] == current_game.board[6])) {
         //  x | x | x       x | - | -
         //  - | - | -       x | - | -
         //  - | - | -       x | - | -
         if (current_game.board[0] == 1) {
            return current_game.host;
         } else if (current_game.board[0] == 2) {
            return current_game.challenger;
         }
      } else if ((current_game.board[2] == current_game.board[5] && current_game.board[5] == current_game.board[8]) ||
                 (current_game.board[6] == current_game.board[7] && current_game.board[7] == current_game.board[8])) {
         //  - | - | -       - | - | x
         //  - | - | -       - | - | x
         //  x | x | x       - | - | x
         if (current_game.board[8] == 1) {
            return current_game.host;
         } else if (current_game.board[8] == 2) {
            return current_game.challenger;
         }
      } else {
         bool is_board_full = true;
         for (uint8_t i = 0; i < board_len; i++) {
            if (is_empty_cell(current_game.board[i])) {
               is_board_full = false;
               break;
            }
         }
         if (is_board_full) {
            return N(draw);
         }
      }
      return N(none);
   }
   ...
}
```
你可以在 [这里](https://github.com/EOSIO/eos/blob/master/contracts/tic_tac_toe/tic_tac_toe.cpp)看到最后的 `tic_tac_toe.cpp`
### 创建 ABI
在这里需要 Abi（又名应用程序二进制接口），这样合约才可以理解您以二进制形式发送的操作。

让我们打开 `tic_tac_toe.abi` 并在此处定义样板：

```json
{
  "types": [],
  "structs": [{
      "name": "...",
      "base": "...",
      "fields": { ... }
  }, ...],
  "actions": [{
      "name": "...",
      "type": "...",
      "ricardian_contract": "..."
  }, ...],
  "tables": [{
      "name": "...",
      "type": "...",
      "index_type": "...",
      "key_names" : [...],
      "key_types" : [...]
  }, ...],
  "clauses: [...]
```
- types：可以由另一个数据结构或内置类型表示的类型列表（想想c / c ++中的typedef）
- structs: 合约中的动作/表使用的数据结构列表
- actions: 合约中可用行动的清单
- tables: 合约中可用的数据表列表

#### 数据表 ABI
请记住，在 `tic_tac_toe.hpp` 中，我们创建一个名为`games`的索引i64表。 它存储游戏结构并使用挑战者作为关键字，其数据类型为account_name。 因此，abi将是：
```json
{
  ...
  "structs": [{
      "name": "game",
      "base": "",
      "fields": [
        {"name":"challenger", "type":"account_name"},
        {"name":"host", "type":"account_name"},
        {"name":"turn", "type":"account_name"},
        {"name":"winner", "type":"account_name"},
        {"name":"board", "type":"uint8[]"}
      ]
    },...
  ],
  "tables": [{
        "name": "games",
        "type": "game",
        "index_type": "i64",
        "key_names" : ["challenger"],
        "key_types" : ["account_name"]
      }
  ],
  ...
}
```

#### 动作 ABI
对于这些动作，我们定义动作内部的动作和结构内动作的结构。
```json
{
  ...
  "structs": [{
    ...
    },{
      "name": "create",
      "base": "",
      "fields": [
        {"name":"challenger", "type":"account_name"},
        {"name":"host", "type":"account_name"}
      ]
    },{
      "name": "restart",
      "base": "",
      "fields": [
        {"name":"challenger", "type":"account_name"},
        {"name":"host", "type":"account_name"},
        {"name":"by", "type":"account_name"}
      ]
    },{
      "name": "close",
      "base": "",
      "fields": [
        {"name":"challenger", "type":"account_name"},
        {"name":"host", "type":"account_name"}
      ]
    },{
      "name": "movement",
      "base": "",
      "fields": [
        {"name":"row", "type":"uint32"},
        {"name":"column", "type":"uint32"}
      ]
    },{
      "name": "move",
      "base": "",
      "fields": [
        {"name":"challenger", "type":"account_name"},
        {"name":"host", "type":"account_name"},
        {"name":"by", "type":"account_name"},
        {"name":"mvt", "type":"movement"}
      ]
    }
  ],
  "actions": [{
      "name": "create",
      "type": "create",
      "ricardian_contract": ""
    },{
      "name": "restart",
      "type": "restart",
      "ricardian_contract": ""
    },{
      "name": "close",
      "type": "close",
      "ricardian_contract": ""
    },{
      "name": "move",
      "type": "move",
      "ricardian_contract": ""
    }
  ],
  ...
}
```

### 编译!
现在我们需要编译tic_tac_toe.cpp来创建tic_tac_toe.wast文件，我们将使用该文件将合约部署到nodeos中。
```bash
$ eosiocpp -o tic_tac_toe.wast tic_tac_toe.cpp
```

### 部署!
现在，wast和abi文件（tic_tac_toe.wast和tic_tac_toe.abi）已准备就绪。 部署时间！

创建一个目录（我们称之为tic_tac_toe）并复制您生成的tic_tac_toe.wast tic_tac_toe.abi文件。

```bash
$ cleos set contract tic.tac.toe tic_tac_toe
```
Ensure that your wallet is unlocked and you have `tic.tac.toe` key imported. If you are going to upload the contract to another account beside `tic.tac.toe`, replace `tic.tac.toe` with your account name and ensure you have the key for that account in your wallet

### Play!
After the deployment and the transaction is confirmed, the contract is already available in the blockchain. You can play with it now!

#### Create
```bash
$ cleos push action tic.tac.toe create '{"challenger":"inita", "host":"initb"}' --permission initb@active 
```
#### Move
```bash
$ cleos push action tic.tac.toe move '{"challenger":"inita", "host":"initb", "by":"initb", "mvt":{"row":0, "column":0} }' --permission initb@active
$ cleos push action tic.tac.toe move '{"challenger":"inita", "host":"initb", "by":"inita", "mvt":{"row":1, "column":1} }' --permission inita@active
```
#### Restart
```bash
$ cleos push action tic.tac.toe restart '{"challenger":"inita", "host":"initb", "by":"initb"}' --permission initb@active 
```
#### Close
```bash
$ cleos push action tic.tac.toe close '{"challenger":"inita", "host":"initb"}' --permission initb@active
```
#### See the game status
```
$ cleos get table tic.tac.toe initb games
{
  "rows": [{
      "challenger": "inita",
      "host": "initb",
      "turn": "inita",
      "winner": "none",
      "board": [
        1,
        0,
        0,
        0,
        2,
        0,
        0,
        0,
        0
      ]
    }
  ],
  "more": false
}
```

