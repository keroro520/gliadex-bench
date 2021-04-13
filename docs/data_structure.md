* A cell:

```
vector Bytes <byte>;
vector BytesVec <Bytes>;

table Cell {
    capacity: uint64;
    data: Bytes;
    witness: Bytes;
    type: Option<Byte32>;
    lock: Byte32;
}
```

* `CKB_Cell` indicates an native token cell, which's `data` and `type` are default empty, `lock` points to `SECP256K1` lock script. Then we represent `CKB_Cell` in the shorthand format:

```
table CKB_Cell {
    capacity: uint64;
}
```

* `UDT_Cell` indicates a sUDT cell, which's `type` points to sUDT's type script, `lock` points to `SECP256K1` lock script. Then we represent `UDT_Cell` in the shorthand format:

```
table UDT_Cell {
    capacity: uint64;
    data:
        udt_amount: uint64;
}
```

`udt_amount` is the owned amount of `UDT` token.

**Note that an `UDT_Cell` is also a `CKB_Cell`, this box contains `UDT` token and `CKB` token.** That means you can spend `CKB` token and `UDT` token from a `UDT_Cell`.

* `Order_Cell` indicates an order cell, which's `type` points to sUDT's type script, `lock` points to `Order`'s lock script. Then we represent `Order`:

```
table Order_Cell {
    capacity: uint64;
    data:
        owned_udt_amount: uint64;
        order_udt_amount: uint64;
        price: uint64;
        order_type: <byte: SELL_CKB | BUY_CKB>;
}
```

`owned_udt_amount` is the owned amount of `UDT` token, `order_udt_amount` is the order amount of `UDT` token, `price` units by `CKB/UDT`, `order_type` can be `SELL_CKB` represents swapping from `CKB` to `UDT`.

`Order_Cell`'s lock is not `SECP256K1`, hence here we don't think it can represent to a native `CKB` token.

* Pending order `CKB => UDT`, named buy-order.

```
// Buy-order `CKB => UDT`

Inputs:
    <vec> CKB_Cell
        Capacity: uint64;
    <vec> UDT_Cell
        Capacity: uint64;
        Data:
            udt_amount: uint64;
Outputs:
    <vec> Order_Cell
        Capacity: sum of inputs ckb capacity;
        Data:
            owned_udt_amount: sum of inputs udt amount;
            order_udt_amount: how many udt want to trade;
            price: uint64;
            order_type: SELL_CKB;
```

* Deal order `CKB => UDT`

Inside a matching transaction, an `Order_Cell` may be matched completely or partially.

```
// Completely Match An Order_Cell
//
// * When an `Order_Cell` been completely matched, its corresponding output is an `UDT_Cell`,
//   or a pure `CKB_Cell` if the `udt_amount` is `0`.
// * Fees are not considered here.

// buy_order.order_type == SELL_CKB
let completely_deal_buy_order = UDT_Cell {
    capacity: buy_order.capacity - buy_order.order_udt_amount * buy_order.price;
    data:
        udt_amount: buy_order.owned_udt_amount + buy_order.order_udt_amount;
}


// sell_order.order_type == BUY_CKB
let completely_deal_sell_order = UDT_Cell {
    capacity: sell_order.capacity + sell_order.order_udt_amount * sell_order.price;
    data:
        udt_amount: sell_order.owned_udt_amount - sell_order.order_udt_amount;
}
```

```
// Partially Match An Order_Cell
//
// * When an `Order_Cell` been partially matched, its corresponding output is an `Order_Cell`.
// * Fees are not considered here.
// * Assume that `partial_udt_amount` dealt.

// buy_order.order_type == SELL_CKB
let partially_deal_buy_order = Order_Cell {
    capacity: buy_order.capacity - partial_udt_amount * buy_order.price;
    data:
        owned_udt_amount: buy_order.owned_udt_amount + partial_udt_amount;
        order_udt_amount: buy_order.order_udt_amount - partial_udt_amount;
        price: buy_order.price;
        order_type: SELL_CKB;
}

// sell_order.order_type == BUY_CKB
let partially_deal_sell_order = UDT_Cell {
    capacity: sell_order.capacity + partial_udt_amount * sell_order.price;
    data:
        owned_udt_amount: sell_order.owned_udt_amount - partial_udt_amount;
        order_udt_amount: sell_order.order_udt_amount + partial_udt_amount;
        price: sell_order.price;
        order_type: SELL_CKB;
}
```

```
// Matching Transaction

Inputs:
    <vec> Order_Cell # buy-order
        Capacity: uint64;
        Data:
            owned_udt_amount: uint64;
            order_udt_amount: uint64;
            price: uint64;
            order_type: SELL_CKB;
    <vec> Order_Cell #  sell-order
        Capacity: uint64;
        Data:
            owned_udt_amount: uint64;
            order_udt_amount: uint64;
            price: uint64;
            order_type: BUY_CKB;
    <...>

Outputs:
    <vec> CKB_Cell
    <vec> UDT_Cell
    <vec> Order_Cell
    <...>
```
