# The Neutron Handoff

This is a proposal to basically approach the most simple integration of Neutron into Qtum possible.

Script Opcodes Added:

* OP_NEUTRON    -- used to enter the Neutron execution space
* OP_NAAL       -- used to track funds owned by the Neutron AAL, internal only. Invalid if not generated by consensus
* OP_NSPEND     -- Used to spend funds owned by the Neutron AAL, internal only. Invalid if not generated by consensus
* OP_NGAS_LIMIT -- Tracks the gas limit and fee rate info for a transaction

This removes most context needs from Qtum and puts it into Neutron. OP_NGAS_LIMIT is used as a simplified method to determine if a transaction can cause gas to exceed the used block gas limit. It may also contain a gas price if an explicit GAS token is not contained. 

All Neutron opcodes used MUST be the first opcode in the transaction script, all data which follows is passed to Neutron and has no consensus rules around it. OP_NSPEND is only valid within a vin script, for spending an OP_NAAL vout. 

## Qtum Script Max Element Size Modification

Our current max element size checking is rather rudimentary and ignores that smart contracts even exist. In order to improve security, there will be a new rule and consensus parameter for script element size. Specifically it'll be the following:

* `constant MAX_ELEMENT_SIZE_NEUTRON_SCRIPT = 0x10000; //64kb`
* `constant MAX_NEUTRON_SCRIPT_SIZE = 0x30000; //196kb`
* All checks for max element size will rather be like so: `if vout.contains_neutron_opcodes() then assert(vout.max_element_size() < MAX_ELEMENT_SIZE_NEUTRON_SCRIPT) else assert(vout.max_element_size() < MAX_SCRIPT_ELEMENT_SIZE) endif`

Note that these limits are only used on the P2P mempool network, and thus modifying them do not require a fork.

## Gas Limit Nesting 

Every transaction containing a vout for OP_NEUTRON must contain an OP_NGAS_LIMIT vout, indicating the total amount of gas possible to be used within this transaction. If the transaction does not contain such a vout, it is invalid and can not be included in a block. If the gas limit described here is less than the gas limit of the OP_NEUTRON vouts (note: parsing of this is hadnled by Neutron), then the top level transaction gas limit will be used. 

So, it is structured more like:

* Neutron vout: "gas limit for this sole execution"
* Gas Limit vout: "absolute gas limit for the entire transaction"

Thus, if there is the following:

* neutron gas limit 2000 -- consumes 1500
* neutron gas limit 4000 -- consumes 3000
* transaction gas limit 5500

The result would be both vouts executing successfully, despite the sum of the gas limit per execution is less than the actual transaction gas limit. 

## Neutron Account Abstraction Layer

In The Handoff implementation, the AAL would be greatly simplified. Every OP_NEUTRON vout with value attached would be spent directly into the AAL. The entire contract system would thus only have 1 UTXO tracking the value of the entire system at the end of each block. There would be only be one condensing transaction per block. The condenscing transaction would also handle gas refunds and Qtum payouts.

### Condensing Transaction and Block Size
The condensing transaction should be rather small, but it's size is impossible to accurately determine until all transactions have been processed. So, block creation code will include a soft rule to only fill blocks up to 5 or 10% of the actual block size limit. If running up against this problem is a common occurence, it could be possible to build a condensing transaction size estimation tool to be run after each transaction. It could never be 100% accurate, but can be used to give an upper bound on what the resulting size would be. 

## Condensing transaction

The condensing transaction is always the last transaction of every block. This transaction spends every OP_NEUTRON and OP_NGAS_LIMIT containing UTXO by using a single OP_NSPEND in the vin's scriptSig. 

The order by which UTXOs are spent is by the order found in the block. The order by which UTXOs are emitted by the condensing transaction is by sorting the overall balance map by NeutronShortAddress. 


# Neutron Interface

The interface to and from Neutron will be fairly simple, consisting of the following Neutron methods. There is no call interface required from Qtum.

Inputs into Neutron:

* add_neutron_transaction -- used for adding tx info for an execution as well as the transaction gas input etc
* add_neutron_execution -- used for adding a new Neutron execution within a UTXO
* add_block_info -- used for adding new block information (ie, block header info)
* create_context -- used to create a new context which can be used for adding new blocks, transactions etc.

Outputs from Neutron:

* get_delta_proof_for_block -- gets the delta proof of any previous block
* parse_gas_fee_info -- parses the gas fee information from an OP_NGAS_FEE UTXO

Executions for Neutron:

* execute_block -- used to execute every execution contained within a block
* commit_context -- used to commit the latest block executed to disk
* destroy_context -- used to abort an in-progress block and not commit it to disk
* revert_top_block -- used for reverting blocks in cases of orphans and reorgs
* partial_execute_queue -- used to execute outstanding transactions/UTXOs. This method only returns block gas consumed and can never be used with `commit_block`. It is used for block creation only.

Outputs from Neutron execution:

* delta_proof
* condensing_transaction_outputs
* block_gas_consumed


Structure for block info:

* Block number (hash is supplied when committed)
* block creators[]
* difficulty
* block time
* consensus type (PoS in the case of Qtum)
* target block spacing (128 currently, potentially 32 in the future)

Structure for txinfo:

* block creator tip (vin - vout)
* vins
    * NeutronAddress
    * Value
* vouts
    * NeutronAddress
    * Value
    * NeutronData[] (only supplied if OP_NEUTRON, unparsed)
* gas limit
* gas price (tbd)

Every operation includes the following info attached:

* chainid -- used for tracking multiple blockchains
* forkid -- used for modifying behavior of Neutron in the case of a fork
* contextid -- used for tracking a top level context. Created  by create_context. Destroyed by commit_context and destroy_context

Protobuf definitions:

    message NeutronAddress{
        uint type;
        bytes data;
    }
    message BlockInfo{
        uint64 height;
        message Creators{
            NeutronAddress address;
            int shares;
        }
        repeated Creators creators;
        int shares_per_block;
        uint64 difficulty;
        uint64 time;
        int consensus_type;
        int target_spacing;
    }

    message TxInfo{
        uint64 block_creator_tip;
        message Vin{
            NeutronAddress address;
            uint64 value;
        }
        repeated Vin vins;
        message Vout{
            NeutronAddress address;
            uint64 value;
            optional bytes neutron_data;
        }
        repeated Vout vouts;
        uint64 gas_limit;
        uint64 gas_price;
    }

    message BlockExecutionResult{
        bytes neutrondb_proof;
        uint64 gas_consumed;
        message CondensingOutputs{
            NeutronAddress address;
            uint64 value;
        }
        repeating CondensingOutputs utxo_outputs;
    }





## Qtum Block Processing Flow

* Validate staking proofs
* use `create_context` to begin a new revertable context
* use `add_block_info` to add block info to Neutron system
* Cycle through and validate each transaction
    * if transaction UTXO script contains OP_NGAS_FEE or OP_NEUTRON, then use `add_neutron_transaction` (once per transaction) followed by `add_neutron_execution` (once per utxo)
* use `execute_block` to execute entire queue of Neutron executions
* create a condensing transaction, spend every OP_NGAS_FEE and OP_NEUTRON UTXO
* with condensing transaction, add vouts based on the resulting `condensing_transaction_outputs` list
* Confirm that the generated condensing txid is the same as the condensing txid contained within the block
* confirm that generated `delta_proof` is the stored `delta_proof`
* Block is accepted, add to database, commit top level Neutron context using `commit_context`

## Qtum Block Creation Flow

* Scan through mempool, add highest fee rate transactions first
* If transaction contains OP_NGAS_FEE, use to determine total gas price and cost. If block has more gas than total gas price, then attempt adding
* Add transaction outputs to Neutron queue. Execute partial_execute_queue. If block gas consumed is less than limit, then add transaction to in progress block. Some creators could do speculative execution to attempt to pack bigger transactions in at the end of the block, but this would require complete re-execution to add any further transactions once the block gas limit is exceeded by the added transaction. It's not reasonably possible to revert a single execution 

## Facilitating block reverting

Using the "reject set" proposal in NeutronDB, there would be an additional dataset stored titled "historic data". It would contain the following data:

* (key_hash, block_number) -> (previous_data, reject_set_number)

In addition to this, the rent data structures would be extended to include "previous historical information". This would be an adjustable set of data which would basically be used in order to prevent needing to rebuilt the entire state structure given a normal amount of orphan blocks and reorgnizations. 


## RPC Integration Design (non-consensus)

The RPC layer of Qtum Core needs to be modified to be capable of generating Neutron transactions. Preferably, the majority of this Neutron knowledge is outsourced, with Qtum Core only receiving blobs of data without needing to understand them. 





