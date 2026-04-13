# Sablier

## Protocol Summary
Sablier is a permissionless token distribution protocol for ERC-20 assets. It can be used for vesting, payroll, airdrops, and more. The sender of a payment stream first deposits a specific amount of ERC-20 tokens in a contract. Then, the contract progressively allocates the funds to the recipient, who can access them as they become available over time. The payment rate is influenced by various factors such as the start time, the end time, the total amount of tokens deposited and the type of stream.

[Ethereum][Solidity][DeFi][Vesting]

### [M-01] Using create opcode is suspicious to re-org attack as it will be deployed on all EVM compatible networks and would result in lost of funds            

#### Summary
The deployment of `SablierV2MerkleLL` and `SablierV2MerkleLT` uses create op-code , This is valunerable to re-orgs Because there could be funds in these contract as the user need to transfer the assets to `SablierV2MerkleLL` and `SablierV2MerkleLT` before creating streams.

#### Vulnerability Details
The Protocol provide the factory contract which is responsible to deploy the `SablierV2MerkleLL` and `SablierV2MerkleLT` contracts for end users to create Airstream Campaign. After deployments of contracts the users nedd to transfer the assets first to these contract after that the user can create new stream via calling claim function. let break this down according to contract flow. here I will focus on  `SablierV2MerkleLL`.

1. User call the `createMerkleLL` function to create new MerkleLockup.
```solidity
function createMerkleLL(
        MerkleLockup.ConstructorParams memory baseParams,
        ISablierV2LockupLinear lockupLinear,
        LockupLinear.Durations memory streamDurations,
        uint256 aggregateAmount,
        uint256 recipientCount
    )
        external
        returns (ISablierV2MerkleLL merkleLL)
    {
        // Deploy the MerkleLockup contract with CREATE.
@>  merkleLL = new SablierV2MerkleLL(baseParams, lockupLinear, streamDurations);

        // Log the creation of the MerkleLockup contract, including some metadata that is not stored on-chain.
        emit CreateMerkleLL(merkleLL, baseParams, lockupLinear, streamDurations, aggregateAmount, recipientCount);
    }
```
2. MerkleLockup will give max approvals to its  core stream type which in this case is Linear. 
```solidity
constructor(
        MerkleLockup.ConstructorParams memory baseParams,
        ISablierV2LockupLinear lockupLinear,
        LockupLinear.Durations memory streamDurations_
    )
        SablierV2MerkleLockup(baseParams)
    {
        LOCKUP_LINEAR = lockupLinear;
        streamDurations = streamDurations_;

        // Max approve the Sablier contract to spend funds from the MerkleLockup contract.
        ASSET.forceApprove(address(LOCKUP_LINEAR), type(uint256).max);
    }
```
3. User transfer the assets to this newly created MerkleLockup contract. this thing is done out side the contract.
4. User will call claim function to create new stream.
```solidity
function claim(
        uint256 index,
        address recipient,
        uint128 amount,
        bytes32[] calldata merkleProof
    )
        external
        override
        returns (uint256 streamId)
    {
...
        // Interaction: create the stream via {SablierV2LockupLinear}.
        streamId = LOCKUP_LINEAR.createWithDurations(
            LockupLinear.CreateWithDurations({
                sender: admin,
                recipient: recipient,
                totalAmount: amount,
                asset: ASSET,
                cancelable: CANCELABLE,
                transferable: TRANSFERABLE,
                durations: streamDurations,
                broker: Broker({ account: address(0), fee: ud(0) })
            })
        );

        // Log the claim.
        emit Claim(index, recipient, amount, streamId);
    }
```
5. The claim function will call `LOCKUP_LINEAR.createWithDurations` function. which will call the `_create` function , `_create` function apart form other things also transfer the assets from `SablierV2MerkleLL` contract to  `LOCKUP_LINEAR` contract.
```solidity
function _create(LockupLinear.CreateWithTimestamps memory params) internal returns (uint256 streamId) {
        ...
        // Load the stream ID.
        streamId = nextStreamId;

        // Effect: create the stream.
        _streams[streamId] = Lockup.Stream({
            amounts: Lockup.Amounts({ deposited: createAmounts.deposit, refunded: 0, withdrawn: 0 }),
            asset: params.asset,
            endTime: params.timestamps.end,
            isCancelable: params.cancelable,
            isDepleted: false,
            isStream: true,
            isTransferable: params.transferable,
            sender: params.sender,
            startTime: params.timestamps.start,
            wasCanceled: false
        });

        // Effect: set the cliff time if it is greater than zero.
        if (params.timestamps.cliff > 0) {
            _cliffs[streamId] = params.timestamps.cliff;
        }

        // Effect: bump the next stream ID.
        // Using unchecked arithmetic because these calculations cannot realistically overflow, ever.
        unchecked {
            nextStreamId = streamId + 1;
        }

        // Effect: mint the NFT to the recipient.
        _mint({ to: params.recipient, tokenId: streamId });

        // Interaction: transfer the deposit amount.
        params.asset.safeTransferFrom({ from: msg.sender, to: address(this), value: createAmounts.deposit });
        ...
    }

```

The following case could occur:
1. User create New `SablierV2MerkleLL` and transfer 100 tokens to it at block 10.
2. user create stream for 50 token at block 11.
3. re-org occur now block 10 gets drop.
4. User will lose his token hold by `SablierV2MerkleLL`. 

#### Impact
As the Smart contract are suppose to compatible with all EVM based chain. Re-org is known issue on Ethereum and other chains. If this happens the user funds locks in contract will be lost.

#### Tools Used
Manual Review

#### Recommendations
Use `CREATE2` op-code with the salt compose of user defined number and `msg.sender`.

---

### [L-01] Stream can not be canceld if transfer are block due to any issue            

#### Summary
The Sabiler Protocol allows the sender to cancel a stream if it is marked as `cancelable`. When the sender cancels the stream, any remaining assets are returned to sender of stream. However, if the assets are paused or the sender is blacklisted, the cancellation will fail. Most of the time, the stream operates based on a schedule. So the recipient will still receives the funds

#### Vulnerability Details
The Sabiler Allows the sender of stream to cancel the stream if it is cancelable. The stream could be canceled if there is some miss agreement between the parties. When sender cancel the Stream the following step execute on smart contract level:

1. Senders calls the cancel function with the `streamId`.
2. The code perform some internal checks to ensure that stream is Ok to cancel.
3. Calculate the remaining amount of stream. we will discuss it in subsequent details.
4. Transfer the assets to sender if any. <i>Here The Issue reside</i>.
5. Hit callback if recipient is smart contract.

`3` here we calculate the amount to send back. The amount for each stream type is calculated using `_calculateStreamedAmount` and `_calculateStreamedAmount` has different implementation for each stream and the common thing is time all the calculation are performed on time basis from `startTime` to `endTime` here I will only explain the `SablierV2LockupLinear`. following code will help to identify the amount withdraw able at given time:
```solidity
190:     function _calculateStreamedAmount(uint256 streamId) internal view override returns (uint128) {
191:         // If the cliff time is in the future, return zero.
192:         uint256 cliffTime = uint256(_cliffs[streamId]);
193:         uint256 blockTimestamp = block.timestamp;
194:         if (cliffTime > blockTimestamp) {
195:             return 0;
196:         }
197: 
198:         // If the end time is not in the future, return the deposited amount.
199:         uint256 endTime = uint256(_streams[streamId].endTime);
200:         if (blockTimestamp >= endTime) {
201:             return _streams[streamId].amounts.deposited;
202:         }
203: 
204:         // In all other cases, calculate the amount streamed so far. Normalization to 18 decimals is not needed
205:         // because there is no mix of amounts with different decimals.
206:         unchecked {
207:             // Calculate how much time has passed since the stream started, and the stream's total duration.
208:             uint256 startTime = uint256(_streams[streamId].startTime);
209:             UD60x18 elapsedTime = ud(blockTimestamp - startTime);
210:             UD60x18 totalDuration = ud(endTime - startTime);
211: 
212:             // Divide the elapsed time by the stream's total duration.
213:             UD60x18 elapsedTimePercentage = elapsedTime.div(totalDuration);
214: 
215:             // Cast the deposited amount to UD60x18.
216:             UD60x18 depositedAmount = ud(_streams[streamId].amounts.deposited);
217: 
218:             // Calculate the streamed amount by multiplying the elapsed time percentage by the deposited amount.
219:             UD60x18 streamedAmount = elapsedTimePercentage.mul(depositedAmount);
220: 
221:             // Although the streamed amount should never exceed the deposited amount, this condition is checked
222:             // without asserting to avoid locking funds in case of a bug. If this situation occurs, the withdrawn
223:             // amount is considered to be the streamed amount, and the stream is effectively frozen.
224:             if (streamedAmount.gt(depositedAmount)) {
225:                 return _streams[streamId].amounts.withdrawn;
226:             }
227: 
228:             // Cast the streamed amount to uint128. This is safe due to the check above.
229:             return uint128(streamedAmount.intoUint256());
230:         }
231:     }
232: 
```
It can be observed from above `LN208` to `LN229` that We first calculate the elapsed time , then take percentage of it with total duration and finally calculate the streamAmount so far. and return the streamAmount.
Now let have a look on Cancel function  code :
```solidity
557:     function _cancel(uint256 streamId) internal {
558:         // Calculate the streamed amount.
559:         uint128 streamedAmount = _calculateStreamedAmount(streamId);
560: 
561:         // Retrieve the amounts from storage.
562:         Lockup.Amounts memory amounts = _streams[streamId].amounts;
563: 
564:         // Check: the stream is not settled.
565:         if (streamedAmount >= amounts.deposited) {
566:             revert Errors.SablierV2Lockup_StreamSettled(streamId);
567:         }
568: 
569:         // Check: the stream is cancelable.
570:         if (!_streams[streamId].isCancelable) {
571:             revert Errors.SablierV2Lockup_StreamNotCancelable(streamId);
572:         }
573: 
574:         // Calculate the sender's amount.
575:         uint128 senderAmount;
576:         unchecked {
577:             senderAmount = amounts.deposited - streamedAmount;
578:         }
579: 
580:         // Calculate the recipient's amount.
581:         uint128 recipientAmount = streamedAmount - amounts.withdrawn;
582: 
583:         // Effect: mark the stream as canceled.
584:         _streams[streamId].wasCanceled = true;
585: 
586:         // Effect: make the stream not cancelable anymore, because a stream can only be canceled once.
587:         _streams[streamId].isCancelable = false;
588: 
589:         // Effect: if there are no assets left for the recipient to withdraw, mark the stream as depleted.
590:         if (recipientAmount == 0) {
591:             _streams[streamId].isDepleted = true;
592:         }
593: 
594:         // Effect: set the refunded amount.
595:         _streams[streamId].amounts.refunded = senderAmount;
596: 
597:         // Retrieve the sender and the recipient from storage.
598:         address sender = _streams[streamId].sender;
599:         address recipient = _ownerOf(streamId);
600: 
601:         // Retrieve the ERC-20 asset from storage.
602:         IERC20 asset = _streams[streamId].asset;  
603: 
604:         // Interaction: refund the sender.
605:         asset.safeTransfer({ to: sender, value: senderAmount }); // <------------------ @audit : it can revert
606: 
607:         // Log the cancellation.
608:         emit ISablierV2Lockup.CancelLockupStream(streamId, sender, recipient, asset, senderAmount, recipientAmount);
609: 
610:         // Emits an ERC-4906 event to trigger an update of the NFT metadata.
611:         emit MetadataUpdate({ _tokenId: streamId });
612: 
613:         // Interaction: if the recipient is a contract, try to invoke the cancel hook on the recipient without
614:         // reverting if the hook is not implemented, and without bubbling up any potential revert.
615:         if (recipient.code.length > 0) {
616:             try ISablierV2Recipient(recipient).onLockupStreamCanceled({
617:                 streamId: streamId,
618:                 sender: sender,
619:                 senderAmount: senderAmount,
620:                 recipientAmount: recipientAmount
621:             }) { } catch { }
622:         }
623:     }

```
After calculation of `streamAmount` we subtract the return `streamAmount` from `amounts.deposited` and also check if there is some assets left for recipient.update the state of a  stream and `refunded` amount. At `LN605` we send the `senderAmount` to sender of stream. It is where the cancel function would revert in case of pause/blacklisting etc.

Let's have a look at  following case : 

1. Sender Bob creates a Linear stream with recipient Alice. 
2. The stream will start from time 1 and ends at time 8.
3. There is some miss agreement between the parties, The Bob wants to cancel the stream at time 3.
4. When Bob submit the cancel transaction The transaction got revert due to any reason i.e transfers are paused/ Bob is blacklisted.
5. Now at time 6 the underline assets transfers are allowed and now Bob cancel the stream. the stream will only send the amount from time 6 to 8.
5. Alice will withdraw the amount from time 3 to 6. which in reality the Alice is not eligible to receive.


#### Impact
The recipient will keep receiving assets if the sender cannot cancel the stream due to transfer issues, even after a missed agreement.

#### Tools Used
Manual Review

#### Recommendations
One mitigation for this issue could be following :
Add struct with following key attributes :
```solidity
struct CancelWithdrawal {
    IERC20 asset;
    address sender;
    uint128 amount;
  }
```
and add mapping of sender to `_cancelWithdrawal`. 
```solidity
    mapping(uint256 id => Lockup.CancelWithdrawal cancelWithdrawal) internal _cancelWithdrawal;
```
add withdrawal function for cancel withdrawal only allow the sender of stream to withdraw.

```solidity
function withdrawCancelAmount(uint256 id, address to) external {
    Lockup.CancelWithdrawal storage withdrawData = _cancelWithdrawal[id];
    if (withdrawData.sender != msg.sender) revert(); // custom error message.
    if (withdrawData.amount > 0) {
      IERC20 asset = withdrawData.asset;

      asset.safeTransfer(to, withdrawData.amount);
      withdrawData.amount = 0;
      // emit event
    }
  }
```

Inside `_cancel` function add following changes:
```solidity
    try    asset.transfer({ to: sender, value: senderAmount }){} catch {// @audit : safeTransfer is internal so can not be used with try catch that's why i will not insist on this solution 
    Lockup.CancelWithdrawal storage withdrawData = _cancelWithdrawal[streamId];
      withdrawData.amount = senderAmount;
      withdrawData.sender = sender;
      withdrawData.asset = asset;
       };

```
For other solution just Keep in Mind that due to some third party issue make sure the cancel stream will not revert. you can also completely remove the transfer from cancel call only update the states and allows the sender to withdraw funds in separate function.

---

### [L-02] if admin is modified then the protocol should allow the new admin to cancel/renounce stream            

#### Summary
The `Adminable` contract allow the current admin to transfer ownership to other address. However there are two issue associated with it 1). The new admin is not able to cancel the stream or renounce stream. 2). The old admin can cancel/renounce the old stream.

#### Vulnerability Details
When ever we create the new stream the current admin is set has a sender of stream.
```solidity
SablierV2MerkleLL.sol:79
79:         // Interaction: create the stream via {SablierV2LockupLinear}.
80:         streamId = LOCKUP_LINEAR.createWithDurations(
81:             LockupLinear.CreateWithDurations({
82:                 sender: admin,
83:                 recipient: recipient,
84:                 totalAmount: amount,
85:                 asset: ASSET,
86:                 cancelable: CANCELABLE,
87:                 transferable: TRANSFERABLE,
88:                 durations: streamDurations,
89:                 broker: Broker({ account: address(0), fee: ud(0) })
90:             })
91:         );

```
At Line `82` the current admin is set as sender of stream. The admin can also be changed to another address
```solidity
Adminable.sol:34
34:     function transferAdmin(address newAdmin) public virtual override onlyAdmin {
35:         // Effect: update the admin.
36:         admin = newAdmin;
37: 
38:         // Log the transfer of the admin.
39:         emit IAdminable.TransferAdmin({ oldAdmin: msg.sender, newAdmin: newAdmin });
40:     }

```
`transferAdmin` function is most of the time used for admin rotation or when admin private key got leaked, Private Key leak is currently 2nd most hacks happen in web3.
if Private key leak happens the team will change the admin to new address and make protocol safe. 
Now let have a look what this admin/sender address is allowed to do.
1. it can cancel the stream. upon canceling the stream amount left will be sent to the sender who has created this stream.
```solidity
function cancel(uint256 streamId) public override noDelegateCall notNull(streamId) {
    // Check: the stream is neither depleted nor canceled.
    ...
    // Check: `msg.sender` is the stream's sender.
    if (!_isCallerStreamSender(streamId)) { // <---------------- old admin only 
      revert Errors.SablierV2Lockup_Unauthorized(streamId, msg.sender);
    }

    // Checks, Effects and Interactions: cancel the stream.
    _cancel(streamId);
  }
```
```solidity
function _cancel(uint256 streamId) internal {
    ...
    // Retrieve the sender and the recipient from storage.
    address sender = _streams[streamId].sender;
    address recipient = _ownerOf(streamId);

    // Retrieve the ERC-20 asset from storage.
    IERC20 asset = _streams[streamId].asset;

    // Interaction: refund the sender.
    // asset.safeTransfer({ to: sender, value: senderAmount });
     asset.safeTransfer({ to: sender, value: senderAmount }); //<---------- asset transfer to the old admin
    ...
  }
```
2. renounce the stream 
```solidity
SablierV2Lockup.sol:261
261:   function renounce(uint256 streamId) external override noDelegateCall notNull(streamId) updateMetadata(streamId) {
262:     // Check: the stream is not cold.
            ....
272:     // Check: `msg.sender` is the stream's sender.
273:     if (!_isCallerStreamSender(streamId)) {
274:       revert Errors.SablierV2Lockup_Unauthorized(streamId, msg.sender);
275:     }
276: 
277:     // Checks and Effects: renounce the stream.
278:     _renounce(streamId);
279: 
        ...
289:   }

```

the Issue here is most of the time the Private key Leaks are handle via changing the admin , here after changing the admin the old one is still able to cancel the stream and steal the funds from stream.
<b>Note: This issue is not added to known Issues , The known issue only state about the callback call. and also the Protocol assumes that all the user will be using safe for admin role which is not correct most of the time.</b>

#### Impact
The Impact will be high if Private key hack occur Because the old admin can still cancel all the stream and steal the funds lock in stream.
#### Tools Used
Manual Review
#### Recommendations
either allow both old and new Admin to cancel and renounce stream or add it known issues. and you can also block the old admin to cancel stream. it is debatable that why i am not suggesting a single fix.



