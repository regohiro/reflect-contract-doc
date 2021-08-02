<style>  
.katex { font-size: 1em !important; }
</style>

# A Technical Whitepaper for Reflect Contracts

## Abstract

a brief summary of RFI 

## Introduction 

The goal is to create a token that will distribute fees to holders when someone makes a transaction. One way is to simply add  their reward to holder's balance, one by one. However, this is not an ideal solution because it could easily run out of gas. A different method is to create a deflationary mechanism so that tokens one holds are worth more. 

also write about excluding stakers

## Background 

Before diving into the contract, a new concept must be introduced: t-space and r-space, along with tTotal and rTotal. tTotal, which belongs to t-space, represents token in circulation or total supply of a token. On the other hand, rTotal, which belongs to r-space, is a reflected value of tTotal. The term "reflected" cannot be easily explained in words but one interpretation of rTotal is total money supply in banks. Furthermore, values in t-space can be easily converted to r-space form, and vice versa. 

### Defining tTotal and rTotal

$$
\begin {alignat}{2}
\mathrm {tTotal} &= \mathrm {totalSupply} \\
\mathrm {rTotal} &= \mathrm {MAX} - (\mathrm {MAX}\bmod \mathrm {tTotal})
\end {alignat}
$$

Where
　• $\mathrm {totalSupply}$ is the total supply of tokens (i.e. 1 billion tokens)
　• $\mathrm {MAX}$ is the maximum number that Solidity can provide ($\approx 10^{77}$) 

$\mathrm {rTotal}$ can be further broken down as follows: 
$$
\begin {align*}
\mathrm {rTotal} &= \mathrm {MAX} - (\mathrm {MAX}\bmod \mathrm {tTotal}) \\
&= \mathrm {MAX} - (\mathrm {MAX} - q \cdot \mathrm {tTotal}) \\
&= q \cdot \mathrm {tTotal} \text{　}  (1\leq q \leq \mathrm {MAX}) 
\end {align*}
$$

From the property of remainders: 
$$
\begin {align*}
0 \leq \mathrm {MAX}\bmod \mathrm {tTotal} < \mathrm {tTotal} 　　\\
-\mathrm {MAX} \leq - q \cdot \mathrm {tTotal} < \mathrm {tTotal} -\mathrm {MAX} \\
\mathrm {MAX} - \mathrm {tTotal} < \mathrm {rTotal}\leq \mathrm {MAX}　　　　 
\end {align*}
$$

Therefore, $\mathrm {rTotal}$ is a multiple of $\mathrm {tTotal}$ and is between  $\mathrm {MAX} - \mathrm {tTotal}$ and  $\mathrm {MAX}$. 

### Conversion

Any t-space value ($t$) can be converted to r-space value ($r$) as follows:
$$
\begin {align}
r = q\cdot t = \frac{\mathrm {rTotal}}{\mathrm {tTotal}} \cdot t
\end {align}
$$


## Transaction

Transaction mechanism is illustrated in the figure below: 

[![image](https://www.linkpicture.com/q/Screen-Shot-2021-08-02-at-11.40.46.png)](https://www.linkpicture.com/view.php?img=LPic61075f154e16f2017733822)
Note: 
　(1) If sender is excluded from staking　　　　　(2) If recipient is excluded from staking 　
Glossary: 
　• tAmount: token amount that sender pays/transfers including the 10% fee 
　• tFee: the 10% transfer fee
　• tTransferAmount: tokens that will get transferred to recipient 
　• tOwned[user]: User's balance represented in t-space (only used by non-stakers)
　• rOwned[user]: User's balance represented in r-space 
　
### Deflationary mechanism 

As illustrated in the figure above, by decreasing rTotal, a deflationary mechanism is achieved. After the first transaction, a newly generated rate will be used for the next transaction or when viewing user balance. This makes tokens in r-space more valuable. 

Relationship of $\textrm{rTotal}$ in n+2, n+1, and n-th transaction: 

$$
\begin{align*}
\textrm{rTotal} (n+1) &= \textrm{rTotal} (n)- \textrm{rFee} (n) \\
&= \left(1 - \frac{\textrm{tFee}(n)}{\textrm{tTotal}}\right) \cdot \textrm{rTotal}(n) \\
\textrm{rTotal} (n+2) &= \textrm{rTotal} (n+1)- \textrm{rFee} (n+1) \\
&= \left(1 - \frac{\textrm{tFee}(n+1)} {\textrm{tTotal}}\right) \cdot \left(1 - \frac{\textrm{tFee}(n)}{\textrm{tTotal}}\right) \cdot \textrm{rTotal}(n)
\end{align*}
$$

Therefore, $\textrm{rTotal}$ in n-th transaction can be calculated using the following formula:

$$
\begin{align}
&\textrm{rTotal}(n) = \prod_{i=0}^{n-1} \left(1- \frac{\textrm{tFee}(n-i)}{\textrm{tTotal}} \right) \cdot \textrm{rTotal}(0)\\
\end{align}
$$

$\textrm{tFee}$ is 10% of tAmount, therefore $\forall \textrm{tFee} \ll \textrm{tTotal}$. This makes tokens in r-space deflationary while maintaining $\textrm{rTotal}$ to be always above 0. 


## Avoiding non-stakers from equation

It is essential to keep non-stakers out of the equation when calculating rate. Instead of using equation from (3) that counts all users, rate is calculated as follows: 

$$
\begin{align}
\textrm{rate} = \frac{\textrm{rTotal} - \sum\limits_{\textrm{user}\in \textrm{non-stakers}} \textrm{rOwned[user]}}{\textrm{tTotal} - \sum\limits_{\textrm{user}\in \textrm{non-stakers}} \textrm{tOwned[user]}} =: \frac{\textrm{rSupply}}{\textrm{tSupply}}
\end{align}
$$

Dev note: Since this operation involves a loop, non-stakers should be no more than 10. 

## Calculating balance

For stakers, balance in t-space is calculated as follows:

$$
\begin{align}
\textrm {balanceOf[user] } = \frac{\textrm {rOwned[user]}}{\mathrm {rate}} = \frac{\textrm {tSupply}}{\textrm {rSupply}} \cdot \textrm {rOwned[user]　　　}  
\end{align}
$$

After a transaction, a new rate will be applied: 
$$
\begin{align*}
\mathrm {balanceOf[user]'} = \frac{\textrm {rOwned[user]}}{\mathrm {rate '}}  = \frac{\textrm {tSupply}}{\textrm {rSupply} - \textrm{rFee}} \cdot \textrm {rOwned[user]} 
\end{align*}
$$

Combining equation (4) and (6), the general formula for balance in n-th transaction is:
$$
\begin{align}
\mathrm{balanceOf[user]}(n) = \frac{\textrm{tSupply}}{\prod\limits_{i=0}^{n-1} \left(1-\frac{\textrm{tFee}(n-i)}{\textrm{tSupply}} \right) \cdot \textrm{rSupply}(0)} \cdot \textrm{rOwned[user]}
\end{align}
$$


For non-stakers, their balance is simply $\textrm{tOwned[user]}$. 

### Calculating using the first method

As mentioned in the introduction, one way to calculate user's balance is to simply add reward to their original balance: 

$$
\begin{align*}
\mathrm {balanceOf[user]'} &= \mathrm {balanceOf[user]} + \mathrm {fee} \cdot \frac{\mathrm {balanceOf[user]}}{\textrm{tSupply} - \textrm{fee}} \\
&=  \frac{\text{tSupply}}{\text{tSupply} - \text{fee}} \cdot \text{balanceOf[user]}
\end{align*}　　　　
$$

However, this method quickly becomes an issue after another transaction: 
$$
\begin{align*}
\mathrm {balanceOf[user]''} &=  \frac{\text{tSupply}}{\text{tSupply} - \mathrm{fee''}} \cdot \mathrm{balanceOf[user] '} \\
&= \frac{\text{tSupply}}{\text{tSupply} - \mathrm{fee''}} \cdot \frac{\text{tSupply}}{\text{tSupply} - \mathrm{fee'}} \cdot \mathrm{balanceOf[user] }
\end{align*}
$$ 

By introducing t-space and r-space, calculating balance becomes simpler. 　

## One issue with RFI contracts

RFI contracts has one fundamental issue as to re-including users into staking. The conversion rate between t-values and r-values is calculated using equation (5). There, it is excluding non-stakers out of equation in order to fully reward the stakers. However, when a user gets re-included into staking, the rate updates, thereby changing all stakers balance. This function can be used to rug-pull rewards from stakers by re-including a whale. 

Note: By default, all users are included in staking. Re-including means to exclude from staking then include after certain period of time. 

## Expanding the contract

Also reward non-native tokens such as BNB, BTCB, CAKE, etc.

## About the author

GitHub: 

