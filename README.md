#ripple-lib-java sucks

In ripple, you'll note that there's a serialization format where there is an
enumeration of fields, each with a unique symbolic string representation
(UpperCase by convention), and also with a `type`, and `nth-of-type` ordinal.

See [here](https://github.com/ripple/rippled/blob/1a7eafb6993f95c4d34e00669a70c8dd4ec0c9ba/src/ripple/module/data/protocol/SField.cpp#L148)

```c++
    SField const sfIndexNext     = make::one(&sfIndexNext,     STI_UINT64, 1, "IndexNext");
    SField const sfIndexPrevious = make::one(&sfIndexPrevious, STI_UINT64, 2, "IndexPrevious");
    SField const sfBookNode      = make::one(&sfBookNode,      STI_UINT64, 3, "BookNode");
```

Serialized `type`s include object types and array types, such that you can make
arbitrarily nested data structures.

The objects are serialized as so:

    $field_bytes$serialized_type_bytes...
    
This is roughly equivalent to:

    {"$key" : "$value"}

Thus in some senses it's much like json, as the structure is somewhat self
describing. (In contrast to a way of encoding structures, where the $keys were
implied by position in a tuple of values)

The type of the $value when serialized, implies the length, for example a
STI_UINT64 is obviously 8 bytes.

We've established that the encoding is similar to JSON, in that it encodes
arbitary structures. It's just that the keys are all predefined.

Now, consider the type ordinal [enumeration](https://github.com/ripple/rippled/blob/1a7eafb6993f95c4d34e00669a70c8dd4ec0c9ba/src/ripple/module/data/protocol/SField.h#L27):

```c++
// // types (common)
STI_UINT16 = 1,
STI_UINT32 = 2,
STI_UINT64 = 3,
STI_HASH128 = 4,
STI_HASH256 = 5,
STI_AMOUNT = 6,
STI_VL = 7,
STI_ACCOUNT = 8,
// 9-13 are reserved
STI_OBJECT = 14,
STI_ARRAY = 15,

// types (uncommon)
STI_UINT8 = 16,
STI_HASH160 = 17,
STI_PATHSET = 18,
STI_VECTOR256 = 19,
```

And consider some of the fields:

```
// 16-bit integers
SField const sfLedgerEntryType = make::one(&sfLedgerEntryType, STI_UINT16, 1, "LedgerEntryType", SField::sMD_Never);
SField const sfTransactionType = make::one(&sfTransactionType, STI_UINT16, 2, "TransactionType");
```

While being 16 bit integers, both fields actually have richer semantics. They
are in fact both enums for various transaction types. 

1. [TransactionType](https://github.com/ripple/rippled/blob/1a7eafb6993f95c4d34e00669a70c8dd4ec0c9ba/src/ripple/module/data/protocol/TxFormats.h#L31)
2. [LedgerEntryType](https://github.com/ripple/rippled/blob/1a7eafb6993f95c4d34e00669a70c8dd4ec0c9ba/src/ripple/module/data/protocol/LedgerFormats.h#L28-29)

The string symbolic representations for TransactionType are defined [here] (https://github.com/ripple/rippled/blob/1a7eafb6993f95c4d34e00669a70c8dd4ec0c9ba/src/ripple/module/data/protocol/TxFormats.cpp#L56) 
and used [here](https://github.com/ripple/rippled/blob/1a7eafb6993f95c4d34e00669a70c8dd4ec0c9ba/src/ripple/module/data/protocol/STParsedJSON.cpp#L206).

Now look at some of the STI_UINT32 fields:

```
// 32-bit integers (common)
SField const sfFlags             = make::one(&sfFlags,             STI_UINT32,  2, "Flags");
SField const sfLedgerSequence    = make::one(&sfLedgerSequence,    STI_UINT32,  6, "LedgerSequence");
...
SField const sfExpiration        = make::one(&sfExpiration,        STI_UINT32, 10, "Expiration");
```

`LedgerSequence` is just a standard number, mapping naturally to std::uint32_t in
C++ or a long in Java (which has no unsigned integer)

`Flags` is a little more interesting; it packs in a bunch of per `TransactionType`
boolean options into an erstwhile `32 unsigned int`. These options have [symbolic names](https://github.com/ripple/ripple
d/blob/1a7eafb6993f95c4d34e00669a70c8dd4ec0c9ba/src/ripple/module/data/protocol/TxFlags.h#L70):

```c++
// OfferCreate flags:
tfPassive
tfImmediateOrCancel
tfFillOrKill
tfSell
```

Finally, `Expiration` is an integer, meaning seconds elapsed since `1 Jan 2000 00:00:00 GMT`, and thus a timestamp.

Imagine a Java class with each of these fields:

``` java
class HodgePodge {
    long flags;
    long close_time;
    long ledger_index;
}
```

It would't lead to the nicest API. Arguably, `close_time` would better be
represented as some kind of Date/LocalDateTime instance, and even the `flags`
could have a nicer API if a richer type was used.

With all that in mind, consider the current implementation of ripple types in
ripple-lib-java. For each of the STI_* enumerated above there's a class
implementing SerializedType:

```java
public interface SerializedType {
    Object toJSON();
    byte[] toBytes();
    String toHex();
    void toBytesSink(BytesSink to);
}
```

In ripple-lib-java all the fields are represented by a `Field` enum class. The
object classes (used to represent a transaction) are essentially wrappers around
`Map<Field, SerializedType>` So, again, this matches the json like ability to
create arbitrary structures, except limited to predefined keys.

Like most JSON apis, the typical api was tried:

    object.getAmount(Field.Amount)

However, what about type safety for put?:

    object.putAmount(Field.LedgerSequence, anAmount)

For this reason, typed fields were enumerated and placed as static members on classes:

    object.get(Amount.LimitAmount)
    object.put(UInt32.LedgerSequence, anAmount) <--- a compile time error

Built on top of these objects (STObject) is a hierarchy for transactions:

    STObject
        Transaction
            Payment
            OfferCreate

The Transaction class has accessors for common transaction fields, and various
methods:

```
public class Transaction extends STObject {
...
    public TransactionType transactionType() {
        return (TransactionType) get(Field.TransactionType);
    }

    public UInt32 flags() {return get(UInt32.Flags);}
    public UInt32 sourceTag() {return get(UInt32.SourceTag);}
    public UInt32 sequence() {return get(UInt32.Sequence);}
    public UInt32 lastLedgerSequence() {return get(UInt32.LastLedgerSequence);}
    public UInt32 operationLimit() {return get(UInt32.OperationLimit);}
    public Hash256 previousTxnID() {return get(Hash256.PreviousTxnID);}
    public Hash256 accountTxnID() {return get(Hash256.AccountTxnID);}
    public Amount fee() {return get(Amount.Fee);}
    public VariableLength signingPubKey() {return get(VariableLength.SigningPubKey);}
    public VariableLength txnSignature() {return get(VariableLength.TxnSignature);}
    public AccountID account() {return get(AccountID.Account);}
    public void transactionType(TransactionType val) {put(Field.TransactionType, val);}
    public void flags(UInt32 val) {put(Field.Flags, val);}
    public void sourceTag(UInt32 val) {put(Field.SourceTag, val);}
    public void sequence(UInt32 val) {put(Field.Sequence, val);}
    public void lastLedgerSequence(UInt32 val) {put(Field.LastLedgerSequence, val);}
    public void operationLimit(UInt32 val) {put(Field.OperationLimit, val);}
    public void previousTxnID(Hash256 val) {put(Field.PreviousTxnID, val);}
    public void accountTxnID(Hash256 val) {put(Field.AccountTxnID, val);}
    public void fee(Amount val) {put(Field.Fee, val);}
    public void signingPubKey(VariableLength val) {put(Field.SigningPubKey, val);}
    public void txnSignature(VariableLength val) {put(Field.TxnSignature, val);}
    public void account(AccountID val) {put(Field.Account, val);}
...
```

While this does lead to nice auto completion, and discoverability of fields, and
is an improvement over using the raw STObject api everywhere, might there not be
a nicer API using POJOS and reflection rather than building objects around a
`Map<Field, SerializedType>` ?

Other than the TransactionType accessors, which is one of the few SerializedType
implementations that doesn't have a corresponding STI_TRANSACTION_TYPE, most of
the types used are generic. Note that `flags()` just returns a `UInt32`, the
standard implementation, with nothing particularly nice about it.

Exhibit le suckery: 

```
public void setCanonicalSignatureFlag() {
    UInt32 flags = get(UInt32.Flags);
    if (flags == null) {
        flags = CANONICAL_SIGNATURE;
    } else {
        flags = flags.or(CANONICAL_SIGNATURE);
    }
    put(UInt32.Flags, flags);
}
```

Perhaps better would be classes like this:

```java
class Payment extends Transaction {
    public Amount paths;
    public AccountID destination;
    public Optional<PathSet> paths;
    public Optional<SendMax> sendMax;
}
```
