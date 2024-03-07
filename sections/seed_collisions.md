# Seed Collisions
## The Vulnerability
Seed collisions occur when different inputs (i.e., seeds and program IDs) used to generate a PDA result in the same PDA address. This
is problematic when PDAs are used within a program for different purposes, as it can lead to unexpected behavior, including denial of
service attacks or complete compromise.

## Example
Consider a program for a decentralized voting platform for various proposals and initiatives. Each voting session for a given proposal or
initiative is created with a unique identifier, and votes are submitted by users. To manage these, the program uses PDAs for both voting
sessions and individual votes:
```rust
// Creating a Voting Session PDA
#[derive(Accounts)]
#[instruction(session_id: String)]
pub struct CreateVotingSession<'info> {
    #[account(mut)]
    pub organizer: Signer<'info>,
    #[account(
        init,
        payer = organizer,
        space = 8 + Product::SIZE,
        seeds = [b"session", session_id.as_bytes()],
    )]
    pub voting_session: Account<'info, VotingSession>,
    pub system_program: Program<'info, System>,
}

// Submitting a Vote PDA
#[derive(Accounts)]
#[instruction(session_id: String)]
pub struct SubmitVote<'info> {
    #[account(mut)]
    pub voter: Signer<'info>,
    #[account(
        init,
        payer = voter,
        space = 8 + Vote::SIZE,
        seeds = [session_id.as_bytes(), voter.key().as_ref()]
    )]
    pub vote: Account<'info, Vote>,
    pub system_program: Program<'info, System>,
}
```
In this scenario, an attacker would try to carefully craft a voting session that, when combined with the static seed `"session"`, would result in a PDA
that coincidentally matches the PDA generated for a different voting session. Deliberately creating a PDA that clashes with another voting session's PDA
could disrupt the platform's operations by, for example, preventing legitimate votes for proposals, or denying new initiatives from being added to the 
platform since Solana's runtime cannot distinguish between the colliding PDAs

## Recommended Mitigation
To mitigate the risk of seed collisions, developers can:
- Use unique prefixes for seeds across different PDAs in the same program. This approach will help ensure that PDAs remain distinct
- Use unique identifiers (e.g., timestamps, user IDs, nonce values) to guarantee that a unique PDA is generated every time
- Programmatically validate a generated PDA does not collide with existing PDAs before executing certain actions