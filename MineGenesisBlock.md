        // Mine genesis block
        {
            // The target PoW (basically how many leading zeros are required)
            arith_uint256 target = arith_uint256().SetCompact(genesis.nBits);
            uint256 uTarget = ArithToUint256(target.GetCompact());

            // The 'best' hash generated so far, with the most leading zeroes
            uint256 uLowest = uint256S("7fffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff");

            while (true) {
                if (genesis.nNonce == 0) {
                    // We have wrapped nNonce, so we will increase nTime and
                    // start over with a nonce of 0 again.
                    std::cerr << "Wrapped nNonce, increasing nTime.\n";

                    genesis.nTime++;
                }

                // Increment nNonce to get a new block hash
                genesis.nNonce++;

                // Did we find a new best?
                uint256 uHash = genesis.GetHash();
                if (UintToArith256(uHash) < UintToArith256(uLowest)) {
                    uLowest = uHash;
                    std::cerr << "New low: " << uLowest.GetHex() << std::endl;
                }

                // Did we create enough PoW?
                if (UintToArith256(uHash) < target) {
                    // Dump the genesis block that we mined
                    std::cerr << "------------------------------------------\n";
                    std::cerr << "Genesis block generated!\n";
                    std::cerr << "Target : " << uTarget.GetHex() << std::endl;
                    std::cerr << "PoW    : " << uHash.GetHex() << std::endl;
                    // Hex should be the same as PoW
                    std::cerr << "Hex    : " << genesis.GetHash().GetHex() << std::endl; 
                    std::cerr << "Gensis block information: \n";
                    std::cerr << "nTime  : " << genesis.nTime<< std::endl;
                    std::cerr << "nNonce : " << genesis.nNonce << std::endl;
                    std::cerr << "Merkle : " << uint256(genesis.hashMerkleRoot).GetHex() << std::endl;
                    std::cerr << "------------------------------------------\n";

                    break;
                }
            }
        }
