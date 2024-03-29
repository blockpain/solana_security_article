## Frontrunning

With the rising popularity of transaction bundlers such as Jito, frontrunning is a concern that should be taken seriously by protocols built on the Solana blockchain. Imagine a protocol that handles purchasing and bidding for a product, storing the sellers pricing information in an account named `SellInfo`. 

```rust
#[derive(Accounts)]
pub struct SellProduct<'info> {
  product_listing: Account<'info, ProductListing>,
  sale_token_mint: Account<'info, Mint>,
  sale_token_destination: Account<'info, TokenAccount>,
  product_owner: Signer<'info>,
  purchaser_token_source: Account<'info, TokenAccount>,
  product: Account<info, Product>
}

#[derive(Accounts)]
pub struct PurchaseProduct<'info> {
  product_listing: Account<'info, ProductListing>,
  token_destination: Account<'info, TokenAccount>,
  token_source: Account<'info, TokenAccount>,
  buyer: Signer<'info>,
  product_account: Account<'info, Product>,
  token_mint_sale: Account<'info, Mint>,
}

#[account]
pub struct ProductListing {
  sale_price: u64,
  token_mint: Pubkey,
  destination_token_account: Pubkey,
  product_owner: Pubkey,
  product: Pubkey,
}
```

To purchase a `Product` that is listed, a buyer would need to pass in the `ProductListing` account related to the product they want. But what if the seller is able to change the `sale_price` of their listing? 

```rust
pub fn change_sale_price(ctx: Context<ChangeSalePrice>, new_price: u64) -> Result<()> {...}
```

This would introduce a frontrunning opportunity for the seller, especially if the buyers purchasing transaction doesnt include `expected_price` checks that would ensure they are paying no more than expected for the product they want. If the purchaser submits a transaction to buy the given `Product` is would be possible for the seller to call `change_sale_price`, and using Jito, ensure this transaction is included before the purchasers transaction. A malicious seller could change the price in the `ProductListing` account to an exorbitant amount, unbeknownst to the purchaser, forcing them to pay much more than expected for the `Product`! 

#### Recommended Mitigation

A simple solution  would be including `expected_price` checks on the purchasing side of the deal, preventing the buyer from paying more than expected for the `Product` they want to buy.

```rust
pub fn purchase_product(ctx: Context<PurchaseProduct>, expected_price: u64) -> Result<()> {
  assert!(ctx.accounts.product_listing.sale_price <= expected_price);
  ...
}

