_The Problem:

User connects a wallet (Phantom) successfully
User signs a transaction (I get the wallet popup and signature)
Transaction is sent (I get a transaction ID)
Transaction fails to confirm with "Transaction confirmation timeout - not found on chain"
When checking the explorer, the transaction is "not found"
This happens consistently, even with different transaction amounts and recipient addresses.

My Environment:

Solana devnet
@solana/web3.js v1.98.0
@solana/wallet-adapter-react
Phantom wallet
Here's my transaction sending code:

  // Get latest blockhash with context - using PROCESSED for fastest 
  blockhash
  const {
    context: { slot: minContextSlot },
    value: { blockhash, lastValidBlockHeight }
  } = await connection.getLatestBlockhashAndContext('processed');

  // Create transaction with priority fee
  const transaction = new Transaction({
    feePayer: publicKey,
    recentBlockhash: blockhash,
  });

  // Add priority fee instruction
  transaction.add(
    ComputeBudgetProgram.setComputeUnitPrice({
      microLamports: 100000 // 0.0001 SOL priority fee
    })
  );

  // Add transfer instruction
  transaction.add(
    SystemProgram.transfer({
      fromPubkey: publicKey,
      toPubkey: recipientPubkey,
      lamports: amount * LAMPORTS_PER_SOL
    })
  );

  // Send transaction with consistent commitment
  const txSignature = await sendTransaction(transaction, connection, {
    minContextSlot,
    skipPreflight: false,
    preflightCommitment: 'processed', // Match the blockhash commitment
    maxRetries: 5
  });

  // Transaction sent! Now confirm it...

  And here's my confirmation code:

  // Wait for confirmation with timeout
  const waitForConfirmation = async (txSignature, blockhash,
  lastValidBlockHeight) => {
    try {
      // Method 1: Use confirmTransaction
      const confirmationPromise = connection.confirmTransaction({
        signature: txSignature,
        blockhash: blockhash,
        lastValidBlockHeight: lastValidBlockHeight
      }, 'confirmed');

      // Method 2: Poll for status
      const statusCheckPromise = new Promise((resolve, reject) => {
        let retries = 0;
        const maxRetries = 30;

        const checkStatus = async () => {
          try {
            // Check if block height has been exceeded
            const currentBlockHeight = await
  connection.getBlockHeight('confirmed');
            if (currentBlockHeight > lastValidBlockHeight) {
              reject(new Error(`Transaction expired: block height 
  exceeded`));
              return;
            }

            const status = await connection.getSignatureStatus(txSignature);
            if (status && status.value) {
              if (status.value.confirmationStatus === 'confirmed' ||
                  status.value.confirmationStatus === 'finalized') {
                resolve(status);
                return;
              }
            }

            retries++;
            if (retries >= maxRetries) {
              reject(new Error('Transaction confirmation timeout - not found
   on chain'));
              return;
            }

            // Continue polling
            setTimeout(checkStatus, 1000);
          } catch (error) {
            setTimeout(checkStatus, 1000);
          }
        };

        checkStatus();
      });

      // Race both methods
      const result = await Promise.race([confirmationPromise,
  statusCheckPromise]);

      // Verify transaction is really on chain
      const txStatus = await connection.getTransaction(txSignature, {
        commitment: 'confirmed',
      });

      if (!txStatus) {
        const status = await connection.getSignatureStatus(txSignature);
        if (!status || !status.value || !status.value.confirmationStatus) {
          throw new Error('Transaction not found on chain');
        }
      }

      return result;
    } catch (error) {
      throw error;
    }
  };
I've tried many approaches:

Increased priority fees
Different commitment levels ('processed', 'confirmed')
Changed blockhash fetching strategies
Added proper error handling and retries
I also see this in my logs: Got blockhash: ABCDE12345... (valid until block 352651718) Current block height: 352651568 Added priority fee: 100000 microlamports Transaction sent: 5g7upHabZCYUjUUmuXATXbt1aMHCqzujvTEqq6WQymD7bcxzkWCE14zW KFfGAM2TWoSL8Km11mY8L1nhCZ8vGcen

Is there something wrong with my approach? Why would the transaction signature be valid but the transaction never shows up on-chain?

javascriptreactjsblockchainweb3jssolana
Share
Edit
Follow
asked 2 days ago
Guy Fernando's user avatar
Guy Fernando
1122 bronze badges
 New contributor
Add a comment
Related questions
3
Best practice of sending multiple dependent transaction [solana] [web3js]
9
Blockhash not found when sending transaction
4
Solana transaction not confirmed
 Load 4 more related questions
Know someone who can answer? Share a link to this question via email, Twitter, or Facebook.
Your Answer
