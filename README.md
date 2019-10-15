# Async payment channel for TON

Our team made a sync channel with all wrappers. However, we did not finished yet with the same wrappers to async channel, we decided to apply only with funC smart contract and docs about it.

# Description

### State 


| Variables |  Size    |       Description           |
|-----------|----------|-----------------------------|
| init      |  1 bit   | flag to initialize contract |
| timeout   | 32 bits  | timeout in second for exit  |
| timestamp | 48 bits  | unix time in seconds        |
| pubkA     | 256 bits | pub key of user A           |
| pubkB     | 256 bits | pub key of user B           |
| fA        |   1 bit  | flag registry for A         |
| fB        |   1 bit  | flag registry for B         |
| X_a       | 214 bits | nanograms registry for A    |
| X_b       | 214 bits | nanograms registry for B    |

The total size is 1023 and it is enough to store in 1 Cell.
214 bits is enough to rotate all balance of TON network for 5e45 times, so, it is safe.

### State evolution

#### Deploy

The first the state is (0, timeout, 0, pubkA, pubkB, 0, 0, 0, 0)

#### Setup

##### Onchain

Users can deposit and withdraw grams and store intermediate values into X_a and X_b.

##### Offchain

Users decide, which amounts __U_a__ and __U_b__ each user needs to deposit to open the channel and which shares __a__ and __b__ will belong to each user as initial.

At first, A sign and send to B `signed(A, (contract_address, b/2))` and B send to A `signed(B, (contract_address, a/2))`, where __c=a+b__

After that A and B sign both and exchanges with`(contract_address, 0, U_a, U_b)`

##### Onchain

Any user send to the contract `(contract_address, U_a, U_b)`, the contract checks, that __X_a__ and __X_b__ more or equal then __U_a__ and __U_b__, after that __init__ flag is set to __1__, remained grams returns to owners, __X_a__ and __X_b__ set to zero.

#### Protocol

Each user stores current value for coins received by another user. When the user needs to send some more value, the user signs new integral value of all amount received by the counterparty as `signed(user, (contract_address, message_code, x))` and send the data to another party.
The procedure is unblockable. Technically, the total user's balance may become negative. We consider negative balance as zero balance.

#### Closing

Let's user A decide to close the channel.

* A send to the protocol tuple of separately signed values `(X_a, X_b)` into the contract. The contract set __X_a__, __X_b__, __timestamp:=now+timeout__, and set __F_b:=1__. This means two things:
    * __F_b__ is fixed and nobody can change it
    * if anybody preset signed by A __F_b__ more then presented by A, the contract send all money to B.

* B can process the same procedure with one restriction. Both values sent by B must be more or equal than the same sent by A.


* If __now > timestamp || F_a && F_b__, the contract provide withdrawal procedure for the users.

#### Withdrawal

```
# Consider C is the total amount, A and B value for users A and B to withdraw

a := min(C, max(0, X_a - X_b + C/2))
b := C - a

```

### Force exit

Send to special method the message (contract_address, exit_code, a), signed by both users, the contract send `a` and `total_amount-a` grams to receivers and close the channel.

# Requirements

- have `func` and `fift` available in `$PATH`
- have ton lite node running
- have wallets for Alice and Bob
- have `alice.addr`, `alice.pk`, `bob.addr`, `bob.pk` in the current dir

# Deployment

- `func stdlib.fc lightning.fc > lightning.fif` - this file is already commited to the repo
- `./deploy.fif <alice addr> <bob addr> <challenge timeout in sec>`
- send some test grams to the displayed address
- in ton node `sendfile deploy.boc`

# Usage

There are a couple of .fif files to work with the smart contract. When executed, they usually write data to `query.boc`, 
which you then need to upload to TON network using `sendfile` in your ton node.

Deposit funds to the payment channel

```
./init.fif # TODO: sign required messages
./deposit.fif alice <channel addr> <amount>
./deposit.fif deposit bob <channel addr> <amount>
```

Now send some funds between alice and bob

```
./send.fif alice bob <amount>
./send.fif send bob alice <amount>
./balance.fif alice
```

You can now close payment channel with fast exit:

```
./fast_exit.fif
```

Or you can use ...


# Notes

See [test](./test) dir for more usage examples. There are tests for each contract function.

# Authors
Nick Kozlov - CTO and Co-founder of BUTTON Wallet (@enormousrage, nk@buttonwallet.com)

Kirill Kuznecov - Co-founder of BUTTON Wallet (@krboktv, kk@buttonwallet.com)

Alexey Prazdnikov - Fullstack developer at BUTTON Wallet (@noprazd, ap@buttonwallet.com)

Max Spiridonov - Lead Backend developer at BUTTON Wallet (@maxSpt, ms@buttonwallet.com)

Ivan Frolov - Backend developer at BUTTON Wallet (@ivnkhh, if@buttonwallet.com)

Roman Semenov - One of founders of Copperbits community, co-author of Tornado.cash Ethereum mixer (@poma, semenov.roma@gmail.com)

Igor Gulamov Blockchain Researcher and Entrepreneur (@igor_gulamov, igor.gulamov@gmail.com)
