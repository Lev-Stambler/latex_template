# Notes
In these notes, I will go through two separate concepts:

1. Elaborating on the "verification wire" which may not be so simply
2. Going through how a BFV ciphertext multiply could be done

<!-- Check out this paper for how to prove low-norm-ness: https://crypto.iacr.org/2022/papers/538804_1_En_3_Chapter_OnlinePDF.pdf -->

## Verification Wire
I spent some time looking at how to "compress" the number of verification wires. Currently, each constraint on the advice needs its own verification wire.

> Example:
>
> An example of a constraint would be checking that a bit decomposition matches the original element. I.e. the constraint equals $\sum_i 2^i B_i - B$ where $\{B_i\}$ is a bit decomposition of $B \in R$. The constraint only passes if the above equals $0$.

We can compress all constraints to check that they all equal zero via using a SIS collision resistant hash function and a low-norm proof. Reading over the low norm proofs though, it seems as if they will be necessary in compressing the input size.

I think a good paper to look at is [LaBRADOR](https://eprint.iacr.org/2022/1341.pdf) which proves a SIS-type relationship.
<!-- 

We want to have some **small constant number** of output wires which can be used to verify if all checks on inputted advice passes. I.e. we do not want to have the number of output verification wires scale linearly with the size of the input advice. Rather, it would be nice if checking that the advice is valid boils down to checking **one or two** output wire.
So, how can we do this? We can use a SIS collision resistant hash function! Lets say we have wire $W$ which should output 0 iff all checks pass. We say that a check value, $X$, passes if $X=0$ and fails otherwise.

First, we will need a helper wire, $H$ that will be used to enforce a "low norm" on all checks $\{X_j\}$. For each check we do:

1. $W_j = h(X_j) + W_{j-1}$
2. $H_j = X_j \cdot H_{j-1}$

where $h$ is a SIS linear hash function (note that this function only needs *linear* operations and thus can be done efficiently with the FC scheme).

Now we need to show two things:

1. $X$ must have small norm in order for the ouput proof to be valid
2. With high probability, $W_{out} = 0$ if and only if $\{X_j\} = 0$.

To show 1, we can note that equation 3.4 of the [FC paper](https://eprint.iacr.org/2022/1368.pdf) can be used to enforce that $X_j$ is placed within the $S_\times$ proof. We can then modify equation 3.5 to handle chaining multiplications together such that $S_\times$ is a single proof for all multiplications on $H_{out}$. 
<!-- TODO: ARRRRRRGHHHHH this is not right!! -->

<!-- Then, now that we know $X_j$ has low norm, an adversary has negligible probability of being able to find $X_j \neq 0$ such that $h(X_j) = W_{j} - W_{j - 1}$ and $W_j = W_{j - 1}$. Also, note that $h(X_j) = 0$ if $X_j = 0$. -->

## Verifying BFV Multiplies
(I have been using [this high level BFV review](https://inferati.azureedge.net/docs/inferati-fhe-bfv.pdf) as reference).

Fortunately, using advice to prove non-arithmetic operations is commonly used in ZK Engineering and we draw inspiration from [circomlib](https://github.com/iden3/circomlib).

Multiplication can be broken down into a few challenging operations:

1. Ring multiplication for $\Z[x] / f$
2. A norm check that the coefficients of $R \in \Z[x] /f$ are in $[0, q)$.
3. Taking ring element $R \in \Z[x] / f$ and putting it to $R \in \Z[x]_q /f$.
4. Ring multiplication for $\Z[x]_q / f$ for relinearization
5. Multiplying by a fraction and rounding to the closest ring element. I.e. $\left\lfloor \frac{t R}{q} \right\rceil$ for ring element $R \in \Z[x] / f$

Let's build up one at a time.

### Ring Multiplication in $\Z[x] / f$
> Summary:
> - $\log_2 q$ pieces of advice
> - $\log_2 q$ multiplications
> - $\log_2 q$ additions
> - $1$ subtraction
> **Total:** $2 \log_2 q + 1$ operations with depth 2 and $\log_2 q$ pieces of advice
In our FC scheme we can work over $\Z[x]_{nq^{2}}/ f$. Thus, we have a logarithmic blowup in $\lambda$ (as n is around $\lambda$) and perform polynomial operations in $\Z[x]_q / f$ without worrying about the modulus in the coefficient ring. Then, to multiply $A, B \in \Z[x]_{q} / f$ lifted to $\Z[x] / f$, we can simply do a multiplication of $A \cdot \left(\sum_i 2^i B_i\right)$ where $\{B_i\}$ is a bit decomposition of $B$ and fed in as advice.

Moreover, the bit decomposition must be verified. It is easy to see that we can simply recover $B$ from the bit decomposition and then use the above outlined verification wire technique to ensure that $B - \sum_i 2^i B_i = 0$. By ring distributive properties, if $B - \sum_i 2^i B_i = 0$ then $A \cdot B - A \sum_i 2^i B_i = 0$. 

### Coefficient Range Constraint
> Summary: 
> - $n \log_2 q$ pieces of advice
> - $n \log_2 q$ subtractions (i.e. $1 - R_{i, j}$)
> - $n \log_2 q$ parallel multiplications for checking constraints
> - $n \log_2 q$ multiplications by constants
> - $n \log_2 q$ additions to recover the original ring element
> - $n \log_2 q$ bound checks for $R_{i, j} \leq \sqrt{nq^2}$
> **Total:** $4 n \log_2 q$ additional operations with depth 4, $n\log_2 q$ norm checks, $n \log_2 q$ pieces of advice

We want to make sure that we can verify whether $R \in \Z[x]_{nq^2} / f$ has coefficients only in $\Z[x]_q$. Unfortunately, I cannot figure out how to do this without a rather expensive amount of advice. Alternatively, we can combine the FC proof system with an external one and have the external proof system check the ranges of $R$. I also think that there is probably a better way to do this. If we could figure out how to do a hadamard ($f \circ g$) product between two ring elements, then we can shorten the advice substantially. Or, some other way to restrict a ring to have coefficients in $\Z_2$ only.

To do the range check we can use a bitwise **and** coefficient wise decomposition. For ring element $R$ we get $\{R_{i, j}\}$ where $i \in 0, ..., \log_2 q - 1$, $j \in 0, ...,n - 1$, and $R_{i, j} \in \Z_2$. Put simply, the $i$ corresponds to the bit decomposition and the $j$ the element wise decomposition.

We can check that $R_{i, j} \in \Z_2$ by adding the constraint that $(R_{i, j}) (1 - R_{i, j}) = 0$. Moreover, we can output $R_{i, j}$ as a constraint wire and check that $R_{i, j} \leq \sqrt{n q^2}$ using low norm testing. Thus, we know that $R_{i, j} \in \Z_2$. Note that if we only consider $i \in 0, ..., \log_2 q - 1$, then we implicitly restrict $R$ to have coefficients less than $q$. Alternatively we can do a check by ensuring that all $R_{i, j}$ for $i \geq \log_2 q$ equal 0.

### Taking the coefficient modulus
> Summary: 
> - 1 constant multiply
> - 1 subtraction to check
> - Requirements of checking "Coefficient Range Constraints"
> - **Total** $4 n \log_2 q + 2$ operations with depth 4, $n\log_2 q$ norm checks, $n \log_2 q$ pieces of advice



As outlined above, the FC scheme can use a coefficient ring of $\Z_{nq^2}$. We then want to take an element from $\Z_{nq^2}$ and place it back into $\Z_q$. Note that $nq \cdot \Z_{nq^2} \cong \Z_{q}$ and that for $x \in \Z_{nq^2}$, $nq \cdot (x \mod q) = nq \cdot x \mod nq^2$. So, to get $[R]_q$ we can provide advice $R' = [R]_q$, check that the coefficients of $R'$ are in $\Z_q$, and then check that $nq R' = nq R \mod nq^2$.
<!-- TODO: VERIFY!!!! -->

### Ring Multiplication in $\Z[x]_q / f$
> Summary:
> - Requirements for 1 ring multiplication and 1 "taking the coefficient modulus"
> **Total** $\log_2 q + 4n \log_2 q + 2$ operations with depth 5, $n \log_2 q$ norm checks, $(n + 1) \log_2 q$ pieces of advice

Say that we have $A, B \in Z[x]_q / f$ and want to get $ A \cdot B = C \in Z[x]_q / f$. Then, we can start with $A, B$ lifted to $Z[x]_{nq^2} / f$. We multiply $A \cdot B$ as in the prior section and then get $C'$. We then use the technique of taking the coefficient modulus as outlined above.

### Multiplying by fraction and rounding to the closest ring element
> Summary:
> - Requirements for "Coefficient range check"
> - $n$ additional pieces of advice for the sign
> - $2n$ additional low norm checks

Here we want to show that, for $A \in \Z[x]_{nq^2} / f$, $A' = \left\lfloor \frac{t A}{q}\right\rceil$.

To multiply by $\frac{t}{q}$ we may need to work over a slightly larger coefficient ring, not $\Z[x]_{nq^2}$ but $\Z[x]_{tnq^2}$. Note that $t \ll q$.
Then, we can get $tA$. Now we have to show that $tA = qA' + R$ where $R$ is the remainder ring element, the coefficients of $R$, $r_j$, satisfy $-q/2 < r_j \leq q/2$, and the coefficients of $qA'$ do not overflow in $\Z[x]_{tnq^2}$.

To do this, we feed the bitwise and coefficient decomposition, $\{R_{i, j}\}$, of $R$ in as advice as well as a "sign bit" for each coefficient, $B_j$. $B_j$ equals 1 if the coefficient on the $j$th term is negative and 0 otherwise. Here, $i \in 0, ..., \log_2 q - 2$ and $j \in 0,..., n-1$.

Then, we use a similar technique as before to constrain $R_{i, j}, B_j \in \Z_2$. We can then prove that 
$$
qA' + R - tA = 0
$$ 
where each coefficient of $r_j$ of R equals
$$
\left(\sum_{i = 0}^{\log_2 q - 2} 2^i R_{i, j}\right) \cdot \left(1 - 2 B_j\right)
$$
Where $R$ is implicitly restrained to have coefficients between $-q/2$ and $q/2$.
We also have to add $n$ low norm checks to ensure that the coefficients of $A'$ do not overflow.

<!-- TODO: FOR TOMORROW SOME MROE EXACT NUBMERS -->

## Overall Cost of BFV Multiplication
To ensure no overflow, we have to work over the coefficient ring $\Z_{2tnq^2}$.
Then, for EvalMult, we need to do
- 4 ring multiplies over $\Z[x]/f$
- 3 multiplying by fractions and rounding to the closest ring element
- and 3 "taking the coefficient modulus"

Then, we, need to relinearize which requires doing:
- 2 ring multiplies
- 2 "taking the coefficient modulus"

Overall one BFV Multiplication and remilitarization requires:
- $12 \cdot \log_2 q + 6$ operations for ring multiplications with depth and $6 \log_2 q$ pieces of advice
- For multiplying by fractions and taking the closest element: $12 n \log_2 q$ additional operations, $3 n\log_2 q + 6n$ norm checks, $3 n \log_2 q$ pieces of advice
- For taking the coefficient modulus, we need $20 n \log_2 q + 10$ operations, $5n\log_2 q$ norm checks, $5n \log_2 q$ pieces of advice

#### Total Summary
One BFV multiplication costs around
- $32 n \log_2 q + 12 \log_2 q + 16$ additional operations
- A depth of 18 (2 for the initial multiplications, 1 for the initial additions, 4 for the fraction multiplication and rounding, 4 for taking the coefficients mod $q$, 2 for relinearization multiplication, 1 for addition, and 4 for taking the coefficients mod $q$).
- $8n \log_2 q + 6n$ norm checks
- $8n \log_2 q + 6 \log_2 q$ pieces of advice


**Remark:** I think that multiplying by $\frac{t}{q}$ and and "taking the coefficient modulus" can be combined into one step (and thus shorten the depth to around 14). I am unsure though.

**Other thoughts:** It may also be worth looking at FHE schemes which do not require modulus switching as that introduces a lot of expense. Maybe Torus FHE? This would require a Torus SIS assumption though.

# Looking at TFHE
