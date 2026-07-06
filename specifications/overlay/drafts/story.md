Story:

1. Node A wants to send to Node Z
2. Node A doesn't know how to send to Node Z
3. Node A constructs $n$ chains, for all $n$ out-neighbors
   - each chain $i$ has the same target ID (labeled $\text{targetID}$), a unique nonce $D_i$, the out-neighbor's designator $m_{i,0}$, and a signature of $\text{Sign}(i, H(\text{targetID}, D_i),m_{i,0})$
