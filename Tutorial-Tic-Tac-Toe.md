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

#### Games Table
For this contract, we will need to have a table that stores a list of games. Let's define it:
```cpp
...
namespace tic_tac_toe {
    ...
    typedef eosio::multi_index< games_account, game> games;
}
  
```
- First template parameter  defines the name of the table
- Second template parameter defines the structure that it stores (will be defined in the next section)

#### Game Structure
Let's define structure for the game. Ensure that this struct definition appears before the table definition in the code.
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
The primary_key method is required by the above table definition for games. That is how the table knows what field is the lookup key for the table.

#### Action Structure
##### Create
To create the game, we need host account name and challenger's account name. The EOSLIB_SERIALIZE macro provides serialize and deserialize methods so that actions can be passed back and forth between the contracts and the nodeos system.
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
##### Restart
To restart the game, we need host account name and challenger's account name to identify the game. Furthermore, we need to specify who wants to restart the game, so we can verify the correct signature is provided.
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
##### Close
To close the game, we need host account name and challenger's account name to identify the game.
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
##### Move
To make a move, we need host account name and challenger's account name to identify the game. Furthermore, we need to specify who makes this move and the movement he is making.
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
You can see the final tic_tac_toe.hpp [here](https://github.com/EOSIO/eos/blob/master/contracts/tic_tac_toe/tic_tac_toe.hpp)

### Main
Let's open tic_tac_toe.cpp and setup the boilerplate
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

#### Action handler
We want tic_tac_toe contract to only react to actions sent to the `tic.tac.toe` account and react differently according to the type of the action. Let's add an impl struct with overloaded 'on' methods taking the different action types (this may seem like overkill for this example, but you will see this pattern employed in other contracts that you can extend, like currency):
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

Notice that we use `unpack_action_data<T>()` before passing it to specific handler, `unpack_action_data<T>()` is converting the action that the contract receives to `struct T`. 

To make things tidy, we will encapsulate the action handlers inside `struct impl`:
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

#### Create Action Handler
For the create action handler, we want to:
1. Ensure that the action has signature from the host
2. Ensure that the challenger and host are not the same player
3. Ensure that there is no existing game
4. Store the newly created game into the db
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

#### Restart Action Handler
For the restart action handler, we want to:
1. Ensure that the action has signature from the host/ challenger
2. Ensure that the game exists
3. Ensure that the restart action is done by host/ challenger
4. Reset the game
5. Store the updated game to the db
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

#### Close Action Handler
For the close action handler, we want to:
1. Ensure that the action has signature from the host
2. Ensure that the game exists
3. Remove the game from the db
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

#### Move Action Handler
For the move action handler, we want to:
1. Ensure that the action has signature from the host/ challenger
2. Ensure that the game exists
3. Ensure that the game is not finished yet
4. Ensure that the move action is done by host/ challenger
5. Ensure that this is the right user's turn
6. Verify movement is valid
7. Update board with the new move
8. Change the move_turn to the other player
9. Determine if there is a winner
10. Store the updated game to the db
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
#### Movement Validation
Valid movement is defined as movement done inside the board on an empty cell:
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
#### Get Winner
Winner is defined as the first player who succeeds in placing three of their marks in a horizontal, vertical, or diagonal row.
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
You can see the final tic_tac_toe.cpp [here](https://github.com/EOSIO/eos/blob/master/contracts/tic_tac_toe/tic_tac_toe.cpp)
### Creating ABI
Abi (a.k.a Application Binary Interface) is needed here, so the contract can understand the action that you send as binary.
Let's open tic_tac_toe.abi and defines the boilerplate here:
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
- types: list of types that can be represented by another data structure or built-in-type (think typedef in c/c++)
- structs: list of data structures used by the action/ table in the contract
- actions: list of actions available in the contract
- tables: list of tables available in the contract

#### Table ABI
Remember that in tic_tac_toe.hpp, we create an single index i64 table called games. It stores `game` structure and use `challenger` as the key, which data type is `account_name`. Hence, the abi will be:
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

#### Actions ABI
For the actions, we define the actions inside `actions` and the structure of the actions inside `structs`.
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

### Compile!
Now we need to compile tic_tac_toe.cpp to create the tic_tac_toe.wast file that we will use to deploy the contract into nodeos. 
```bash
$ eosiocpp -o tic_tac_toe.wast tic_tac_toe.cpp
```

### Deploy!
Now the wast and abi files (tic_tac_toe.wast and tic_tac_toe.abi) are ready. Time to deploy!
Create a directory (let's call it tic_tac_toe) and copy your generated tic_tac_toe.wast tic_tac_toe.abi files.
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

