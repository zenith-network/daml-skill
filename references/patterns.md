# Daml Patterns

Use this file only when you need syntax reminders, examples, or a compact list of Daml-specific pitfalls.

## Greenfield Conventions

When a project has no established Daml style yet, prefer:

- one primary template per module unless two templates are tightly coupled
- singular template names
- `<Template>_<Choice>Result` for result records
- small payload `data` types instead of deeply nested anonymous tuples
- a `script do` example for each non-trivial workflow
- interfaces only when multiple templates need the same callable surface

## Template Skeleton

```daml
template Asset
  with
    issuer : Party
    owner : Party
    amount : Decimal
  where
    signatory issuer
    observer owner
    ensure amount > 0.0

    choice Transfer : ContractId Asset
      with
        newOwner : Party
      controller owner
      do create this with owner = newOwner
```

## Result Record Pattern

Prefer this over large tuples for non-trivial choices.

```daml
data Payment_AcceptResult = Payment_AcceptResult with
  lockedAssetCid : ContractId LockedAsset
  changeCid : Optional (ContractId Asset)

choice Payment_Accept : Payment_AcceptResult
  with
    inputs : [TransferInput]
  controller payer, walletProvider
  do
    lockedAssetCid <- create ...
    pure Payment_AcceptResult with
      lockedAssetCid
      changeCid = None
```

## Interface Skeleton

```daml
data TokenView = TokenView with owner : Party

interface Token where
  viewtype TokenView
  getAmount : Int

template Coin
  with
    issuer : Party
    owner : Party
    amount : Int
  where
    signatory issuer
    observer owner

    interface instance Token for Coin where
      view = TokenView with owner
      getAmount = amount
```

Useful operations:

- `toInterface @Token coin`
- `fromInterface @Coin ifaceValue`
- `toInterfaceContractId @Token coinCid`
- `fetchFromInterface @Coin tokenCid`

## Contract Key Skeleton

```daml
data AssetKey = AssetKey with
  issuer : Party
  symbol : Text

template Asset
  with
    issuer : Party
    symbol : Text
    owner : Party
  where
    signatory issuer
    observer owner
    key AssetKey issuer symbol : AssetKey
    maintainer key.issuer
```

Remember:

- by-key operations are authorization-sensitive
- visibility alone does not imply by-key authorization
- empty maintainer sets and malformed keys are real failure cases

## Exception And Rollback

```daml
exception InvalidTransfer with reason : Text
  where
    message "InvalidTransfer: " <> reason

choice SafeTransfer : ContractId Asset
  with
    newOwner : Party
  controller owner
  do
    try do
      assertMsg "Owner unchanged" (newOwner /= owner)
      create this with owner = newOwner
    catch
      (InvalidTransfer _) -> throw (InvalidTransfer "transfer failed")
```

Use exceptions only when rollback behavior is semantically meaningful.

## Daml Script Test Skeleton

```daml
testTransfer = script do
  issuer <- allocateParty "Issuer"
  alice <- allocateParty "Alice"
  bob <- allocateParty "Bob"

  cid <- submit issuer do
    createCmd Asset with issuer, owner = alice, amount = 10.0

  cid2 <- submit alice do
    exerciseCmd cid Transfer with newOwner = bob

  submitMustFail issuer do
    exerciseCmd cid2 Transfer with newOwner = issuer
```

Use:

- `submit` for success paths
- `submitMustFail` for expected failure
- `trySubmit` when the exact error type matters

## Common Daml-Specific Pitfalls

- A party knowing a `ContractId` does not imply visibility.
- Non-consuming exercises do not automatically disclose created contracts to template observers.
- Choice observers can act as deliberate disclosure edges.
- `ensure` is checked on create and can matter again during implicit upgrades.
- Changing signatories, observers, or interface views can break upgrade compatibility.
- `fetch` by contract ID and `fetchByKey` have different authorization and visibility behavior.
- Making a party an observer instead of a signatory is often an availability decision, not just a privacy decision.
