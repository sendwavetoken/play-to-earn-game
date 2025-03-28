sendwave-p2e/
├── app/           # Frontend code
├── programs/      # Solana programs
│   └── sendwave-p2e/
│       ├── src/
│       │   └── lib.rs  # Program logic
│       └── Cargo.toml
├── tests/         # Program tests
└── migrations/    # Deployment scripts
use anchor_lang::prelude::*;
use anchor_spl::token::{self, Token, TokenAccount, Transfer};

declare_id!("YourProgramIDWillGoHere");

#[program]
pub mod sendwave_p2e {
    use super::*;

    // Initialize the game
    pub fn initialize_game(ctx: Context<InitializeGame>) -> Result<()> {
        let game_account = &mut ctx.accounts.game_account;
        game_account.admin = *ctx.accounts.admin.key;
        game_account.reward_token = ctx.accounts.reward_token.key();
        game_account.reward_amount = 100; // Default reward amount
        Ok(())
    }

    // Player completes a task to earn tokens
    pub fn complete_task(ctx: Context<CompleteTask>) -> Result<()> {
        let game_account = &ctx.accounts.game_account;
        
        // Transfer reward tokens to player
        let cpi_accounts = Transfer {
            from: ctx.accounts.game_token_vault.to_account_info(),
            to: ctx.accounts.player_token_account.to_account_info(),
            authority: ctx.accounts.game_account.to_account_info(),
        };
        
        let cpi_ctx = CpiContext::new(
            ctx.accounts.token_program.to_account_info(),
            cpi_accounts,
        );
        
        token::transfer(cpi_ctx, game_account.reward_amount)?;
        
        emit!(TaskCompleted {
            player: *ctx.accounts.player.key,
            amount: game_account.reward_amount,
        });
        
        Ok(())
    }

    // Admin can update reward amount
    pub fn update_reward(ctx: Context<UpdateReward>, new_amount: u64) -> Result<()> {
        require!(
            *ctx.accounts.admin.key == ctx.accounts.game_account.admin,
            ErrorCode::Unauthorized
        );
        
        ctx.accounts.game_account.reward_amount = new_amount;
        Ok(())
    }
}

#[derive(Accounts)]
pub struct InitializeGame<'info> {
    #[account(init, payer = admin, space = 8 + 32 + 32 + 8)]
    pub game_account: Account<'info, GameAccount>,
    #[account(mut)]
    pub admin: Signer<'info>,
    /// CHECK: This is the token mint we'll be rewarding with
    pub reward_token: AccountInfo<'info>,
    #[account(
        init,
        payer = admin,
        token::mint = reward_token,
        token::authority = game_account,
    )]
    pub game_token_vault: Account<'info, TokenAccount>,
    pub token_program: Program<'info, Token>,
    pub system_program: Program<'info, System>,
}

#[derive(Accounts)]
pub struct CompleteTask<'info> {
    #[account(mut)]
    pub game_account: Account<'info, GameAccount>,
    #[account(mut)]
    pub game_token_vault: Account<'info, TokenAccount>,
    #[account(mut)]
    pub player_token_account: Account<'info, TokenAccount>,
    pub player: Signer<'info>,
    pub token_program: Program<'info, Token>,
}

#[derive(Accounts)]
pub struct UpdateReward<'info> {
    #[account(mut)]
    pub game_account: Account<'info, GameAccount>,
    #[account(mut)]
    pub admin: Signer<'info>,
}

#[account]
pub struct GameAccount {
    pub admin: Pubkey,
    pub reward_token: Pubkey,
    pub reward_amount: u64,
}

#[event]
pub struct TaskCompleted {
    pub player: Pubkey,
    pub amount: u64,
}

#[error_code]
pub enum ErrorCode {
    #[msg("Unauthorized")]
    Unauthorized,
}
import { Program, AnchorProvider, web3 } from '@project-serum/anchor';
import { PublicKey, SystemProgram, Token } from '@solana/web3.js';
import { SendwaveP2e } from './types/sendwave_p2e';
import idl from './idl/sendwave_p2e.json';

// Initialize the program
const programId = new PublicKey('YourProgramID');
const provider = new AnchorProvider(connection, wallet, {});
const program = new Program<SendwaveP2e>(idl as any, programId, provider);

// Initialize the game
async function initializeGame(rewardTokenMint: PublicKey) {
  const [gameAccount, bump] = await PublicKey.findProgramAddress(
    [Buffer.from("game")],
    program.programId
  );
  
  const vault = await getAssociatedTokenAddress(
    rewardTokenMint,
    gameAccount,
    true
  );
  
  await program.methods.initializeGame()
    .accounts({
      gameAccount,
      admin: wallet.publicKey,
      rewardToken: rewardTokenMint,
      gameTokenVault: vault,
      tokenProgram: TOKEN_PROGRAM_ID,
      systemProgram: SystemProgram.programId,
    })
    .rpc();
}

// Player completes a task
async function completeTask(gameAccount: PublicKey, rewardTokenMint: PublicKey) {
  const [vault, _] = await PublicKey.findProgramAddress(
    [Buffer.from("game")],
    program.programId
  );
  
  const playerTokenAccount = await getAssociatedTokenAddress(
    rewardTokenMint,
    wallet.publicKey
  );
  
  await program.methods.completeTask()
    .accounts({
      gameAccount,
      gameTokenVault: vault,
      playerTokenAccount,
      player: wallet.publicKey,
      tokenProgram: TOKEN_PROGRAM_ID,
    })
    .rpc();
}

// Admin updates reward amount
async function updateReward(gameAccount: PublicKey, newAmount: number) {
  await program.methods.updateReward(newAmount)
    .accounts({
      gameAccount,
      admin: wallet.publicKey,
    })
    .rpc();
}
anchor build
anchor deploy
