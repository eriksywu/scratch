# Objective

Find a mapping $DS$ (cumulative downsampler) such that 
$$
DS : {X} \longrightarrow {Y} \times {R}, \quad where \quad {X} =  \mathbb{R}^N, \quad  {Y} =  \mathbb{R}^M, 
\quad M \le N,\quad N \gt 1, \quad M \gt 1 \quad \text{and} \quad M \text{ is a cost to minimize.}
$$

$DS$ has these constraints:
1. $ \exists {m} \; in \; {M} \; \text{such that } Increase(X) = Increase(D(X)) \; \text{for all } x \in X $
    - The increase observed in the input and output must be equal
2. $\forall x \in X, \; y = DS(x) \; => y_1 = x_1 \; and \; y_m = x_n$
    - The first and last values of the input must be preserved as-is in the output. This preserves reset points between blocks.
    - Preserving in this case that all properties of the value is preserved, including if whether the datapoint is a reset (either via value or ST).
        - authors note: this point here can be more rigourously defined but for the same of this exercise, we can take it mean that if $x_{N}$ is a reset point in $x$, it is also a reset point in $y$
    


$ Increase $ can be defined in code as:
```
func increase(x []datapoint) number {
    var adjustment number
    for i := 1; i < len(x); i++ {
        if isReset(x[i-1], x[i]) {
            adjustment += x[i-1].Value()
        }
    }
    return x[len(x)-1].Value() - x[0].Value() + adjustment
}
```
## Additional Defnitions/Lemmas

Generally, if $x$ is a sequence of cumulative datapoints with 0 or more resets, we can transform $x$ as $x'$ for which: \
(1) ${x'_i} = {x_i} + \sum_{k=1}^{i} adjust(x,k)$ such that
$ adjust(x', i) = 0 \text{ for all }i$ 
- aka a cumulative rebuild where all resets in $x$ have been adjusted for in $x'$
  ` x= [1,2,1,3,1] => x'=[1,2,1+2,3+2,1+2+3]=[1,2,3,5,6]`


(2) $adjust(x,i) = \text{the reset adjustment for the ith value in x}$ where \
$adjust(x,k) \ge 0 \; (0 \; if x_k \text{ is not a reset, } x_{k-1} \; if x_k \text{ is a reset} ) $ 
- note: we ignore the definition of what a reset here so we don't rely on the absolute values of any point in our analysis

For any $x$, the increase from the first to the ith point is \
(3) $increase(x, i) = x_i - x_1  + \sum_{k=1}^{i} adjust(x,k)$ 

(4) Thus: $increase(x, i) = increase(x', i)$ which is trivial to prove: \
LHS: $ x_{N} - x_{1} + \sum_{k=1}^{N} adjust(x, k) $\
RHS: $ x'_{N} - x'_{1} + \cancel{\sum_{k=1}^{N} adjust(x',k)} = {x_N} + \sum_{k=1}^{N} adjust(x, k) - x_1  $

## Base Solution: rebuild entire block as cumulative

Let the $DS$ be a function that for any $x$, it outputs $y=[x_1, x'_{N}]$

Obviously this preserves the $increase$ constraint but does not preserve the last datapoint. So not a viable solution.


## Attempt one: rebuild block as cumulative except for last datapoint
A variation of option 1 but the last element of  $x$ is kept as-is as the last element of the output $y$


More generally for any $N \ge 3$ we can choose $ M = 3$ where $y = [x_1, x'_{N-1}, x_{N}]$ 

However this solution does not work for all $x$. 

A simple proof by contradiction:

### Assume: $increase(x, N) = increase(y, M)$  for all $x$ 
#### LHS:  $ x_{N} - x_{1} + \sum_{k=1}^{N} adjust(x, k) $\
#### RHS:  $ x_{N} - x_{1} + adjust([x'_{N-1}, x_{N}]) $


case 1: $x_N$ is a reset point \
$ adjust([x'_{N-1}, x_{N}]) = x'_{N-1} = x_{N-1} + \sum_{k=1}^{N-1} adjust(x, k) =\sum_{k=1}^{N} adjust(x, k) $ \
thus LHS = RHS 

case 2: $x_N$ is not a reset point \
$ adjust([x'_{N-1}, x_{N}]) = 0 $ \
 but $\sum_{k=1}^{N} adjust(x, k) = 0$ is a contradiction for any $x$ where a reset point is present at or before $x_{N-1}$

A simple real example of case 2:
```
x = [4, 1, 2] 
increase(x) = 2 - 4 + 4 = 2
downsampleKeepLast(x) = [4, 5, 2]
increase(downsampleKeepLast(x)) = 2 - 4 + 5 = 3
```



## Solution Rebuild block until last unbroken sequence
We can observe that in option 2 if $x_{N}$ itself is a reset then the increase values between the input and output are equal. 
An example:

```
x = [4, 3, 2] 
increase(x) = 2 - 4 + 4 + 3 = 5 
downsampleKeepLast(x) = [4, 7, 2]
increase(downsampleKeepLast(x)) = 2 - 4 + 7 = 5
```

We can show that a generalized downsampler algorithm that rebuilds the input as a cumulative until the last reset will satifiy all constraints.



Assume the last reset point in $x$ is at $r$, then we propose an algorithm that downsamples $x$ to $y=[x_1, x'_{r-1}, x_r, x_N]$ 

$increase(x) = x_N - x_1 +  \sum_{k=1}^{r} adjust(x, k) = x_N - x_1 +  x_{r-1} + \sum_{k=1}^{r-1} adjust(x, k)$ \
$increase(y) = x_N - x_1 + x'_{r-1} =x_N - x_1 + x_{r-1} + \sum_{k=1}^{r-1} adjust(x, k)$

$qed$

The special case where the last reset is the last datapoint has been proven above. 

