# Block execution parallelization
## Abstract
The main challenge when implementing concurrent execution in different areas of CS is the data contentions occurring between executors. The only way to make parallelization safely is to manage such contentions and avoid them as much as possible, though when it is not possible to avoid them the only solution is to serialize execution. 

We should keep in mind, that the problem can’t be solved with correct synchronization in executor code alone. 
Consider simple example, imagine block with two transactions: 
```
Balances: (A = 10 B = 0) 
1: A->B = 10 (transfer 10) 
2: B->C = 5 (transfer 5)
```
These transactions can’t be executed in parallel, since they have contention on B's balance. If this transactions will be executed concurrently, one part of nodes can execute it in straight order and another will reorder execution which will cause network split. (These transactions can be parallelized though if we allow negative balances, and it can work in some cases, but in for more complex contentions, eg with smart contracts, this approach won’t help)

## Transaction parallelization
We should differ block creation and block verification, since in the former case proposer can choose any correct order, but in the letter all honest verifiers must get exactly the same result.

Consider creating 2 address sets (read set and write set). Each record in such a set is the MPT node this transaction interacts with.
Note that transaction can operate on every record in the subtree, we will call such a node a range (eg contract data addresses ADDRESS + *). 

Further I will propose how to safely break transactions in independent subsets that can be executed in arbitrary order without breaking any safety assumptions.

### Execution paths creation algorithm
#### Transfers
There are several options where to place Rsets and Wsets, let's put them in different structure in the block header as ``map[transaction_hash]tuple<Rset,Wset>``

There are 3 tx types in 0 chain, for TxnTypeSmartContract, TxnTypeData algorithm is simpler: 
```go 
err = sctx.AddTransfer(state.NewTransfer(txn.ClientID, txn.ToClientID,state.Balance(txn.Value)))
```  
for every transfer put from address to Rset and to address to Wset.

#### Smart contracts
SC has transfers (which can be handled the same way as we do for other transaction types) and additionally they operate on MPT nodes that store specific data, which is the most challenging part of parallelization. Collections stored in MPT and serialized as single node are the most difficult part. Smart contract can calculate aggregation functions, eg count elements and use result somewhere in the code later. For now we will keep things simple: every change in collections adds all elements to Wset, every read adds collection node to Rset.

For example: Interestpoolsc has  Pools ``` map[datastore.Key]*interestPool `json:"pools"` ```

MinerNodes is a collection serialized as a node 
```go
func getNodesList(balances cstate.StateContextI, key datastore.Key) (*MinerNodes, error) {
	nodesBytes, err := balances.GetTrieNode(key)
	if err != nil {
		return nil, err
	}

	nodesList := &MinerNodes{}
	if err = nodesList.Decode(nodesBytes.Encode()); err != nil {
		return nil, err
	}

	return nodesList, nil
}
}
```
#### How to create block? 
Let’s keep this process sequential and don’t add any preemptive optimizations. 
1. Get transaction from the tx_pool
2. Apply it to block’s state
3. Track every state access and modification (similar to state changes) and create R/Wsets. Ideally all this logic should be incapsulated in state DB. It will be extremely useful if we can extract all changes to MPT as a sequence of events and filter changes from this sequence. 

As next optimization step, we can try to collect transactions in parallel, when we determine contention, we can choose one transaction and rerun other contenders. Really, we can do it this way, since we only need to find any non erroneous execution path, but not the one when we calidate block.

### Execution paths validation algorithm
We should follow several rules here:
1. Transactions with sets without intersections can be executed in arbitrary order
2. Transactions with only intersections on R-sets can be executed in arbitrary order
3. Intersections in any set with Wset creates happens-before relation. Other transactions wich has intersection with Wset (no matter on Wset or Rset) are not allowed to be reordered and mast be executed sequentially.

Consider simple mark algorithm O(n*n):
1. Each transaction is wrapped with Rset, Wset and Rank:
```go
type TxWrapped struct {
	Rank int
	Rset map[int]bool
	Wset map[int]bool
	Balances cstate.StateContextI
	Tx *Transaction
}
```
2. Visit all transactions further in the tx_list, if there are contention mark transaction to be executed after current
```go
for i, tx := range txs {
	for j := i; j < len(txs); j++ {
		if txs[j].HasContention(tx.Wset) { //If this transaction has contention on R or W set
			if txs[j].Rank <= tx.Rank { // If this transaction is not before current, move it further in rank list
				txs[j].Rank = tx.Rank + 1
			}
		}
	}
}
```
3. Prepare needed state for each transaction based on R/Wsets
4. Group buckets, slice them for parallel factor and execute
```go buckets := make(map[int][]Tx) //make buckets grouped by rang
for i, tx := range txs{
	b := buckets[tx.Rank]
	buckets[tx.Rank] = append(b, tx)
}

for every bucket in parralel {  
    tx.Balances = GetNeededState(globalState, tx.Rset, tx.Wset)
    calculatedStates[I] := calculateState(tx.balances, tx)  //calculate state for each transaction
    compare(tx.Rset, calculatedStates[I].Rset) //validate execution path
    compare(tx.Wset, calculatedStates[I].Wset) //validate execution path
  }
  MergeState(globalState, calculatedStates…) //if no errors apply partialStates to global and finish execution
```

## Attack vectors
1. Malicious proposer can create sets that cause chain split due to ignoring contentions.
Imagine malicious leader who decides no to include some addresses to R/Wsets, honest validators will execute transactions in parallel with contention. Due to not definitive nature of parallelization, validators can split in several groups with different states each. 
Solution: calculate R/Wsets independently for each transaction execution during validation and compare them with given in the block.



