# How default assume valid & checkpoints work in bitcoin

To understand how assume valid & checkpoints work in bitcoin there are a few variables to first learn about:

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

<br />
<br />
<br />


### ```std::map<int, uint256> MapCheckpoints```

https://github.com/bitcoin/bitcoin/blob/0.19/src/chainparams.cpp#L139-L155

```
checkpointData = {
  {
    { 11111, uint256S("0x0000000069e244f73d78e8fd29ba2fd2ed618bd6fa2ee92559f542fdb26e7c1d")},
    { 33333, uint256S("0x000000002dd5588a74784eaa7ab0507a18ad16a236e7b1ce69f00d7ddfb5d0a6")},
    { 74000, uint256S("0x0000000000573993a3c9e41ce34471c079dcf5f52a0e824a81e7f953b8661a20")},
    {105000, uint256S("0x00000000000291ce28027faea320c8d2b054b2e0fe44a773f3eefb151d6bdc97")},
    {134444, uint256S("0x00000000000005b12ffd4cd315cd34ffd4a594f430ac814c91184a0d42d2b0fe")},
    {168000, uint256S("0x000000000000099e61ea72015e79632f216fe6cb33d7899acb35b75c8303b763")},
    {193000, uint256S("0x000000000000059f452a5f7340de6682a977387c17010ff6e6c3bd83ca8b1317")},
    {210000, uint256S("0x000000000000048b95347e83192f69cf0366076336c639f9b7228e9ba171342e")},
    {216116, uint256S("0x00000000000001b4f4b433e81ee46494af945cf96014816a4e2370f11b23df4e")},
    {225430, uint256S("0x00000000000001c108384350f74090433e7fcf79a606b8e797f065b130575932")},
    {250000, uint256S("0x000000000000003887df1f29024b06fc2200b55f8af8f35453d7be294df2d214")},
    {279000, uint256S("0x0000000000000001ae8c72a0b0c301f67e3afca10e819efa9041e458e9bd7e40")},
    {295000, uint256S("0x00000000000000004d9b4ef50f0f9d686fd69db2e03af35a100370c64632a983")},
  }
};
```

```MapCheckpoints``` is a list of block heights along with the block hash at that height. This allows node to check that the block they download at a certain height matches the known hash at the height in the list of checkpoints.

These variables are all set and updated by the Bitcoin Core developers. Usually they are updated with new major releases, sometimes more frequently. 

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

