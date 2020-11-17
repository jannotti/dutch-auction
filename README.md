# A Dutch Auction for a new Algorand Standard Asset (ASA)

Suppose you create an ASA on the Algorand blockchain.  How can you get
the ASA into the hands of interested users?  Perhaps the fairest
method is to hold a Dutch Auction.  In fact, Algorand distributed the
first Algos this way.

https://algorand.foundation/algo-auctions

In Algorand's first auctions for algos, the bids were placed on the
blockchain, but processing of the bids to determine winners was
performed off-chain before algos were eventually distributed on-chain.

Today, using Smart Contracts, the entire auction can be conducted
on-chain. This document will explain how you can use TEAL, Algorand's
Layer-1 Smart Contract Language, to implement a Dutch Auction for a
new ASA.  For simplicity, we'll conduct the auction using Algos as the
bidding currency, but the technique would be quite similar for any
ASA - perhaps a stablecoin like USDC or USDT would be more appropriate
for your auction.


1. Create your ASA.  In this document, we'll call it NewCoin.

1. Create ESCROW\_ACCT where you'll place AUCTION\_SIZE NewCoins.

1. Create a Stateful TEAL contract, which we'll call the AUCTIONEER.
   The AUCTIONEER will be configured with an OPENING\_ROUND,
   OPENING\_PRICE, DEADLINE, and RESERVE\_PRICE.

1. Wait for calls from bidders to the AUCTIONNEER.

1. Any time after the auction has closed (defined below) bidders make
   a second call to the auctioneer to claim their NewCoins and any
   remaining Algos.


We'll show all of the steps that need to be taken by you, as the
issuer of NewCoin, and by bidders with command-line invocations of
`goal`, Algorand's tool for creating, signing, and submitting
transactions to the ALgorand Blockchain.  You'll want to get yourself
setup to run `goal` by following one the many getting started
guides. In initial testing, the _sandbox_ connected to _testnet_ is
probably the easiest way to try things out. Choose a more
production-ready setup before conducting auctions for real money on
_mainnet_.

## Create your ASA

An [ASA](https://developer.algorand.org/docs/features/asa/) is an
Algorand Standard Asset. On the Algorand blockchain, new tokens are
created as "first class citizens," and supported directly by normal
transactions - even atomic swaps of multiple assets among multiple
accounts in a single step.  Read our (tutorial)[] for more details
about creating your ASA, or follow these steps to get going
immediately.  We will assume you begin with a funded account, that
we'll refer to as `SOURCE_ACCT`.

```
./sandbox goal asset create --creator $SOURCE_ACCT --total 1000 --unitname NEWCOIN --decimals 0
Please enter the password for wallet '$SOURCE_WALLET':
Issued transaction from account $SOURCE_ACCT, txid IEFN4GXHDMGM73PMPCRKZTAVCHM2547HXBA2G74XZCM2QNW3LBSA (fee 1000)
Transaction IEFN4GXHDMGM73PMPCRKZTAVCHM2547HXBA2G74XZCM2QNW3LBSA still pending as of round 10382797
Transaction IEFN4GXHDMGM73PMPCRKZTAVCHM2547HXBA2G74XZCM2QNW3LBSA committed in round 10382799
Created asset with asset index 13089202
```

Let's set a variable based on that outcome for use later when we need to specify the asset
id.
```
NEWCOIN=13089202
```

## Create and Fund the ESCROW\_ACCT

Create a new account to hold the NEWCOIN for auction, and the Algos that
will be bid.

```
./sandbox goal account new
Please enter the password for wallet '$SOURCE_WALLET':
Created new account with address ZE3BWLIJNBB6G4CI63MPAWMRN2RYJEIU7Z47JEE6TQIWSXUJJZOH5NXFKA
```

Let's set a variable to hold the ESCROW\_ACCT

```
ESCROW_ACCT=ZE3BWLIJNBB6G4CI63MPAWMRN2RYJEIU7Z47JEE6TQIWSXUJJZOH5NXFKA
> ./sandbox goal account balance -a $ESCROW_ACCT
0 microAlgos
```

The new account will need to be funded.  All accounts must have at
least 100,000 microAlgos.  Further, we soon seen this account needs
more than that, so let's give it 350,000 microAlgos.

```
> ./sandbox goal clerk send -a 350000 -t $ESCROW_ACCT -f $SOURCE_ACCT
Please enter the password for wallet 'jj-sandbox':
Sent 350000 MicroAlgos from account $SOURCE_ACCT to address $ESCROW_ACCT, transaction ID: YKO6RUZQULKAXS2NEKLV7OHAD2QO54AOCV4AUTKBVV5SUI3SERLA. Fee set to 1000
Transaction YKO6RUZQULKAXS2NEKLV7OHAD2QO54AOCV4AUTKBVV5SUI3SERLA still pending as of round 10384814
Transaction YKO6RUZQULKAXS2NEKLV7OHAD2QO54AOCV4AUTKBVV5SUI3SERLA
committed in round 10384816

> ./sandbox goal account balance -a $ESCROW_ACCT
350000 microAlgos
```

The new account must be opted in to the auctionable token. Opting into
an ASA increases an account's minimum balance by 100,000 microAlgos. (You'll
also need to opt into the bid token if you decide to use an ASA other
than algos.)

Opt in by sending 0 of the new coin to the ESCROW\_ACCT, from the ESCROW\_ACCT.
```
./sandbox goal asset send -a 0 --assetid  $NEWCOIN -t $ESCROW_ACCT -f $ESCROW_ACCT
Please enter the password for wallet 'jj-sandbox':
Issued transaction from account ZE3BWLIJNBB6G4CI63MPAWMRN2RYJEIU7Z47JEE6TQIWSXUJJZOH5NXFKA, txid JRS2QRZ4QR3BKLEBKMW73NUL64IIXLC4KFKF3DWA467I27J644UA (fee 1000)
Transaction JRS2QRZ4QR3BKLEBKMW73NUL64IIXLC4KFKF3DWA467I27J644UA still pending as of round 10384984
Transaction JRS2QRZ4QR3BKLEBKMW73NUL64IIXLC4KFKF3DWA467I27J644UA committed in round 10384986
```


Fund the account with the NEWCOIN for the auction.  Having created
1000 NEWCOIN in SOURCE\_ACCT, let's prepare to auction 100 of them
by sending 100 NEWCOIN to ESCROW\_ACCT.

```
./sandbox goal asset send -a 100 --assetid  $NEWCOIN -t $ESCROW_ACCT -f $SOURCE_ACCT
Please enter the password for wallet 'jj-sandbox':
Issued transaction from account LCKVRVM2MJ7RAJZKPAXUCEC4GZMYNTFMLHJTV2KF6UGNXUFQFIIMSXRVM4, txid JOVU4IJSOBWPZV7BOV5VTEGDMQKTJEIZEKBHTEHEOBQ6HDLWQ5RQ (fee 1000)
Transaction JOVU4IJSOBWPZV7BOV5VTEGDMQKTJEIZEKBHTEHEOBQ6HDLWQ5RQ still pending as of round 10385078
Transaction JOVU4IJSOBWPZV7BOV5VTEGDMQKTJEIZEKBHTEHEOBQ6HDLWQ5RQ committed in round 10385080
```


## Create the AUCTIONEER

The next step is to create a TEAL _Application_. On Algorand, TEAL can
be used as a stateless "predicate language" that is inserted along
with a transaction to conduct simple tests of the blockchain state
before allowing the transaction to proceed.  However, it can also be
used in a "stateful" manner.  TEAL can be associated with an address to
create an _application_.  Later, transactions can _call_ that stateful
application. Again, the TEAL either allows or denies the transaction,
but added power comes from an application's ability to read and right
additional state.  The application has _global_ state, and _local_
state.  The _global_ state is associated with the application itself.
The _local_ state is replicated - each address that opts in to the
application has a copy of the _local_ state which the application can
use to track per account information.

Like all data on the blockchain, all of this state is publicly
readable, but the state can only be _written_ by the application.
Thus it is a safe place for the application to store information about
the bids it receives and the amounts that need to be paid out.

The basic idea of the Dutch Auction TEAL application is to accept
bids during the auction writing a "note to itself" in application
_local_ state about the bids coming from each address, and the sum of
all outstanding bids in _global_ state.  When the total of the bids is
sufficient to buy the tokens for sale (recall that in a Dutch Auction,
the price is constantly decreasing, so the ability of the bid total to
buy the tokens constantly grows) or when the auction ends by deadline,
it becomes possible to determine each bidders win share and (perhaps)
refund.

The file (dutch_auction.sh) is a shell script that prints TEAL
scripts customized by several variables documented at the top.  The
variables and TEAL are each heavily commented to help you run your
auction. Be VERY CAREFUL with changes.

If you've set the various variables, you can generate the TEAL code by
downloading the script and running it as `bash dutch_auction.sh >
da_approval.teal 2> da_clear.teal`.  This run creates two scripts -
all TEAL applications consists of an _approval_ program and a
_clear state_ program. Next, create the _application_.

```
./sandbox goal app create --creator $SOURCE_ACCT  --approval-prog da_approval.teal --clear-prog da_clear.teal --global-byteslices 1 --global-ints 1 --local-byteslices 1 --local-ints 1
Please enter the password for wallet 'jj-sandbox':
Attempting to create app (approval size 62, hash SQSMDTYSXQQSFZPG63BE62I5TYCONBV5PHATCR3O6BJDX75PMFXQ; clear size 5, hash U4TBITSQWSG2G4IMJYBOJPEEC535YPRSFMR2JE3ZMXDGY27U3CCA)
Issued transaction from account LCKVRVM2MJ7RAJZKPAXUCEC4GZMYNTFMLHJTV2KF6UGNXUFQFIIMSXRVM4, txid H6RKU2OVSVA5TQ2B4C6QFF3YK6ATWNPDFYE2ZJA5T2OVDPYPUINQ (fee 1000)
Transaction H6RKU2OVSVA5TQ2B4C6QFF3YK6ATWNPDFYE2ZJA5T2OVDPYPUINQ still pending as of round 10407938
Transaction H6RKU2OVSVA5TQ2B4C6QFF3YK6ATWNPDFYE2ZJA5T2OVDPYPUINQ committed in round 10407940
Created app with app index 13101908
```

Once again, let's set a shell variable for future use:
```
AUCTIONEER=13101908
```

## Await BIDS

Starting in BEGIN\_ROUND, bidders can begin making _commitments_ to
buy NewCoins by calling the AUCTIONEER, and paying their bid into
escrow with ESCROW\_ACCT.  This call record the commitment in _local_
state associated with the AUCTIONEER and the bidding account.

To make bids, interested parties will need an account that has opted
in to the AUCTIONEER application.  To demonstrate, we will create two
accounts, ALICE and BOB; fund them; and opt them into the AUCTIONEER.

```
> ./sandbox goal account new
Please enter the password for wallet 'jj-sandbox':
Created new account with address BX65TTOF324PU3IU5IXZ6VFUX3M33ZQ5NOHGLAEBHF5ECHKAWSQWOZXL4I
> ALICE=BX65TTOF324PU3IU5IXZ6VFUX3M33ZQ5NOHGLAEBHF5ECHKAWSQWOZXL4I
> ./sandbox goal account new
Created new account with address TDCYVRHYNTAMZVEOIIGWQPU6GYVWOH5JGYBRFM63MALVKMJQXT66IY3RAE
> BOB=TDCYVRHYNTAMZVEOIIGWQPU6GYVWOH5JGYBRFM63MALVKMJQXT66IY3RAE
```

A bid consists of two operations.  The AUCTIONEER must be informed of
the bid, and the funds associated with the bid must be moved into
ESCROW\_ACCOUNT. Algorand supports _transaction groups_ in which
multiple transactions are submitted to the blockchain in a group. A
bid will consist of an application call transaction in the first slot,
and a pay transaction in the second. All transactions of a group share
the same fate --- they either all succeed of all fail.  Furthermore,
TEAL code for any transaction can examine the details of all others in
the group.  Therefore, the call to the AUCTIONEER can gate the auction
however you might like.  Maybe only certain accounts can bid, maybe
bids must exceed a certain minimum.  If the AUCTIONEER rejects the
bid, then the associated payment into escrow will not happen.  Our
AUCTIONEER is quite permissive though, and will accept all comers!  If
the first argument to the AUCTIONNEER is "bid", the AUCTIONEER
confirms that the second transaction in the group is a `pay` to the
ESCROW\_ACCOUNT and uses the size of the payment as the size of the
bid.

[Need to write about bid levels: Not sure whether we will require
bidders to explicitly state their bid level, since the price curve
will be known, so if they bid, that can mean they are bidding at a
high enough level.]


To construct a bid from ALICE, in which she is willing to spend 200
microAlgos (at the current auction price level or lower):

```
```


## Perform payouts (and refunds)

[combine the power authorization power of stateless TEAL with the
accounting power of stateful TEAL
applications](https://developer.algorand.org/articles/linking-algorand-stateful-and-stateless-smart-contracts/)

One of the unusual aspects of programming for a Blockchain is that
code never runs "spontaneously."  You won't have a TEAL program that
runs when the auction completes, paying out the winners and refunding
the losers.  Instead, the AUCTIONEER has maintained records of which
bids win and which lose, and once the auction closes, those bidders
will close out their position by making a call to the AUCTIONEER to
receive their winnings and/or their refund.  (One bid may be a winner
and a loser! Since there's a finite amount of NewCoin up for auction,
the final bids that come in may win some NewCoin, but not as much as
they had bid on.)  Since TEAL is an _Approval Language_, bidders will
construct a transaction that requests their winnings and refunds, and
the AUCTIONEER will approve those transactions, allowing the bidders
to get their tokens from the ESCROW\_ACCT.



# Next time

Once you've issued and auctioned your new tokens, you might want to
maintain a liquid market for exchanging those tokens with a currency
like Algos, or a stablecoins.  We'll look at how TEAL can be used to
implement a uniswap-like mechanism to allow those exchanges without
human intervention.
