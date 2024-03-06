# Seed Collisions
## The Vulnerability
Seed collisions occur when different inputs (i.e., seeds and program IDs) used to generate a PDA result in the same PDA address. This
is problematic when PDAs are used within a program for different purposes, as it can lead to unexpected behavior, including denial of
service attacks or complete compromise.

## Example
Consider a program for an auction platform where products and bids are tracked using PDAs. A product's PDA is generated with a static 
string (i.e., `"product"`) and the product name as its seeds whereas a bid's PDA could include the product name and the bidder's public 
key as its seeds:
```rust
// Creating a Product PDA
#[derive(Accounts)]
#[instruction(product_name: String)]
pub struct CreateNewProduct<'info> {
    #[account(mut)]
    pub user: Signer<'info>,
    #[account(
        init,
        payer = user,
        space = 8 + Product::SIZE,
        seeds = [b"product", product_name.as_bytes()],
    )]
    pub product: Account<'info, Product>,
    pub system_program: Program<'info, System>,
}

// Creating a Bid PDA
#[derive(Accounts)]
#[instruction(product_name: String)]
pub struct CreateNewBid<'info> {
    #[account(mut)]
    pub user: Signer<'info>,
    #[account(
        mut,
        seeds = [b"product", product_name.as_bytes()],
        bump = product.bump
    )]
    pub product: Account<'info, Product>,
    #[account(
        init,
        payer = user,
        space = 8 + Bid::SIZE,
        seeds = [product_name.as_bytes(), user.key().as_ref()]
    )]
    pub bid: Account<'info, Bid>,
    pub system_program: Program<'info, System>,
}
```
In this scenario, an attacker would try to carefully craft a product that, when combined with the static seed `"product"` would result in a PDA that
coincidentally matches the PDA generated for a bid on a different product. Deliberately creating a PDA that clashes with another product's PDA could
disrupt the platform's operations by, for example, preventing legitimate bids from being placed on certain products or denying new products from being 
added since Solana's runtime cannot distinguish between the colliding PDAs.

## Recommended Mitigation
To mitigate the risk of seed collisions, developers can:
- Use unique prefixes for seeds across different PDAs in the same program. This approach will help ensure that PDAs remain distinct
- Use unique identifiers (e.g., timestamps, user IDs, nonce values) to guarantee that a unique PDA is generated every time