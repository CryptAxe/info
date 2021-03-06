# How default assume valid works in bitcoin

To understand how assume valid works in bitcoin there are a few variables to first learn about:

### ```uint256 defaultAssumeValid```

https://github.com/bitcoin/bitcoin/blob/0.19/src/chainparams.cpp#L89-L90

```        
// By default assume that the signatures in ancestors of this block are valid.
consensus.defaultAssumeValid = uint256S("0x00000000000000000005f8920febd3925f8272a6a71237563d78c2edfdd09ddf"); // 597379
```

```defaultAssumeValid``` is the hash of a block for which ancestors (all previous blocks connecting to this one) are assumed to be valid.

<br />
<br />
<br />

### ```uint256 nMinimumChainWork```

https://github.com/bitcoin/bitcoin/blob/0.19/src/chainparams.cpp#L86-L87

```
// The best chain should have at least this much work.
consensus.nMinimumChainWork = uint256S("0x000000000000000000000000000000000000000008ea3cf107ae0dec57f03fe8");
```

```nMinimumChainWork``` is a 256 bit representation of the amount of work that the best chain should have at minimum. This is a way of represententing the sum of all work in the blocks up to a certain block height, which is updated frequently. The point of this is to prevent new nodes from downloading a chain that is valid but isn't actually the bitcoin chain with the most work. By setting and updating this number with the sum of work on the current chain, nodes can tell whether the chain they are about to download has at least the amount of work that is expected. 


These variables are set and updated by the Bitcoin Core developers. Usually they are updated with new major releases, sometimes more frequently. 

<br />
<br />
<br />


# How assume valid works:


https://github.com/bitcoin/bitcoin/blob/0.19/src/validation.cpp#L1947-L1976

```
bool fScriptChecks = true;
if (!hashAssumeValid.IsNull()) {
    // We've been configured with the hash of a block which has been externally verified to have a valid history.
    // A suitable default value is included with the software and updated from time to time.  Because validity
    //  relative to a piece of software is an objective fact these defaults can be easily reviewed.
    // This setting doesn't force the selection of any particular chain but makes validating some faster by
    //  effectively caching the result of part of the verification.
    BlockMap::const_iterator  it = m_blockman.m_block_index.find(hashAssumeValid);
    if (it != m_blockman.m_block_index.end()) {
        if (it->second->GetAncestor(pindex->nHeight) == pindex &&
            pindexBestHeader->GetAncestor(pindex->nHeight) == pindex &&
            pindexBestHeader->nChainWork >= nMinimumChainWork) {
            // This block is a member of the assumed verified chain and an ancestor of the best header.
            // Script verification is skipped when connecting blocks under the
            // assumevalid block. Assuming the assumevalid block is valid this
            // is safe because block merkle hashes are still computed and checked,
            // Of course, if an assumed valid block is invalid due to false scriptSigs
            // this optimization would allow an invalid chain to be accepted.
            // The equivalent time check discourages hash power from extorting the network via DOS attack
            //  into accepting an invalid block through telling users they must manually set assumevalid.
            //  Requiring a software change or burying the invalid block, regardless of the setting, makes
            //  it hard to hide the implication of the demand.  This also avoids having release candidates
            //  that are hardly doing any signature verification at all in testing without having to
            //  artificially set the default assumed verified block further back.
            // The test against nMinimumChainWork prevents the skipping when denied access to any chain at
            //  least as good as the expected chain.
            fScriptChecks = (GetBlockProofEquivalentTime(*pindexBestHeader, *pindex, *pindexBestHeader, chainparams.GetConsensus()) <= 60 * 60 * 24 * 7 * 2);
        }
    }
}

```

<br />

Take a moment to read the comments in the code above. You should have a basic understanding of what is going on here based on the comments. In summary, if a block is an ancestor of the block with the hash matching ```defaultAssumeValid``` and the block is part of a chain with at least ```nMinimumChainWork``` the scripts in the block will not be verified. As of Bitcoin Core version 0.19 this means that blocks prior to 597379 will not have the scripts in the block verified.

<br />

In more detail, ```bool fScriptChecks``` is set to true. Then the software checks if there is a hashAssumeValid set. If set, the software checks whether the block is an ancestor (came before) the block with the hash matching ```hashAssumeValid``` and whether the block is part of a chain with at least ```nMinimumChainWork```. If all of that is true, the software then uses the function ```GetBlockProofEquivalentTime``` which calculates the amount of time it would take to redo the work of the block we are deciding whether to check the scripts of all the way to the chain tip, with the current estimated amount of network hashrate. 

https://github.com/bitcoin/bitcoin/blob/0.19/src/chain.cpp#L137-L152

```
int64_t GetBlockProofEquivalentTime(const CBlockIndex& to, const CBlockIndex& from, const CBlockIndex& tip, const Consensus::Params& params)
{
    arith_uint256 r;
    int sign = 1;
    if (to.nChainWork > from.nChainWork) {
        r = to.nChainWork - from.nChainWork;
    } else {
        r = from.nChainWork - to.nChainWork;
        sign = -1;
    }
    r = r * arith_uint256(params.nPowTargetSpacing) / GetBlockProof(tip);
    if (r.bits() > 63) {
        return sign * std::numeric_limits<int64_t>::max();
    }
    return sign * r.GetLow64();
}
```

If all of the checks mentioned before are true, and the block proof equivalent time is greater than 60 * 60 * 24 * 7 * 2 (two weeks in seconds) then ```fScriptChecks``` will be set to false. When ```fScriptChecks``` is set to false, the scripts from the block will not be added to the validation queue for verification. 

https://github.com/bitcoin/bitcoin/blob/0.19/src/validation.cpp#L2085

```CCheckQueueControl<CScriptCheck> control(fScriptChecks && nScriptCheckThreads ? &scriptcheckqueue : nullptr);```

