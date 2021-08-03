
<style>  
.katex { font-size: 1em !important; }
</style>

# A Technical Whitepaper for Reflect Contracts

## Abstract

The goal is to create a token that will distribute fees to holders when someone makes a transaction. One way is to simply add reward to users balance, one by one, using a loop. However,  such transaction could easily fail due to network gas limit. A different method is to create a deflationary mechanism so that tokens one holds are worth more. 

## Introduction 

Over the couple of month, auto-staking (auto-farming) tokens have become a new trend in the DeFi space. Some call this the DeFi 2.0 token. 

In traditional farms, users have to stake their tokens or manually. For some users, this feels like a two-step process after supplying liquidity to pools or swapping to native token. Moreover, the staked token will end up in a smart contract, and some users fear of getting rug pulled. 

Reflect contract, or RFI contract, solve this issue by implementing an auto-staking feature built in to the token. Also, one can easily exclude users from staking to avoid whales taking all reward. However, one big difference from traditional farm is that reward comes from transaction fees and not from MasterChef minting new tokens.    

## Background 

Before diving into the contract, a new concept must be introduced: t-space and r-space values, along with tTotal and rTotal. tTotal, which belongs to t-space, represents tokens in circulation or total supply of a token. On the other hand, rTotal, which belongs to r-space, is a reflected value of tTotal. The term "reflected" cannot be easily explained in words but one interpretation of rTotal is token supply in reserve. Furthermore, values in t-space can be easily converted to r-space form, and vice versa using formula (3). 

Stakers are users who earn passive income by holding native token. In contrast, non-stakers do not earn rewards. Router contracts, pair contracts, dev wallets are usually excluding from staking in order to fully reward users. 

### Defining tTotal and rTotal

$$
\begin {alignat}{2}
\mathrm {tTotal} &= \mathrm {totalSupply} \\
\mathrm {rTotal} &= \mathrm {MAX} - (\mathrm {MAX}\bmod \mathrm {tTotal})
\end {alignat}
$$

$\mathrm {rTotal}$ can be further broken down as follows: 
$$
\begin {align*}
\mathrm {rTotal} &= \mathrm {MAX} - (\mathrm {MAX}\bmod \mathrm {tTotal}) \\
&= \mathrm {MAX} - (\mathrm {MAX} - q \cdot \mathrm {tTotal}) \\
&= q \cdot \mathrm {tTotal} \text{　}  (1\leq q \leq \mathrm {MAX})
\end {align*}
$$

Moreover, from the property of remainders: 
$$
\begin {align*}
0 \leq \mathrm {MAX}\bmod \mathrm {tTotal} < \mathrm {tTotal} 　　\\
-\mathrm {MAX} \leq - q \cdot \mathrm {tTotal} < \mathrm {tTotal} -\mathrm {MAX} \\
\mathrm {MAX} - \mathrm {tTotal} < \mathrm {rTotal}\leq \mathrm {MAX}　　　　 
\end {align*}
$$

Therefore, from the two equations above, $\mathrm {rTotal}$ is a multiple of $\mathrm {tTotal}$ and is between  $\mathrm {MAX} - \mathrm {tTotal}$ and  $\mathrm {MAX}$. For the value $\mathrm {MAX}$, many choose the biggest number that Solidity provides which is ~uint256(0) $\approx 10^{77}$. 

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
　• tAmount: token amount that sender pays/transfers including tFee 
　• tFee: the 10% transfer fee (note: 10% was chosen arbitrarily) 
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
&\textrm{rTotal}(n) = \prod_{i=1}^{n} \left(1- \frac{\textrm{tFee}(n-i)}{\textrm{tTotal}} \right) \cdot \textrm{rTotal}(0)\\
\end{align}
$$

$\textrm{tFee}$ is usually around 10% of tAmount, therefore $\forall \textrm{tFee} \ll \textrm{tTotal}$. This makes tokens in r-space deflationary while maintaining $\textrm{rTotal}(\forall n) > 0$. 


## Avoiding non-stakers from equation

It is essential to keep non-stakers out of the equation when calculating rate. Instead of using equation from (3) that counts all users, rate is calculated as follows: 

$$
\begin{align}
\textrm{rate} = \frac{\textrm{rTotal} - \sum\limits_{\textrm{user}\in \textrm{non-stakers}} \textrm{rOwned[user]}}{\textrm{tTotal} - \sum\limits_{\textrm{user}\in \textrm{non-stakers}} \textrm{tOwned[user]}} =: \frac{\textrm{rSupply}}{\textrm{tSupply}}
\end{align}
$$

Dev note: Since this operation involves a loop, non-stakers should be no more than 10 and is recommended to store the summation part in storage variables. 

## Calculating balance

For stakers, balance in t-space is calculated as follows:

$$
\begin{align}
\textrm {balanceOf[user] } = \frac{\textrm {rOwned[user]}}{\mathrm {rate}} = \frac{\textrm {tSupply}}{\textrm {rSupply}} \cdot \textrm {rOwned[user]　　　}  
\end{align}
$$

After a transaction, a new rate will be applied: 
$$
\begin{align}
\mathrm {balanceOf[user]'} = \frac{\textrm {rOwned[user]}}{\mathrm {rate '}}  = \frac{\textrm {tSupply}}{\textrm {rSupply} - \textrm{rFee}} \cdot \textrm {rOwned[user]} 
\end{align}
$$

Therefore, the general formula for balance in n-th transaction is:
$$
\begin{align}
\mathrm{balanceOf[user]}(n) = \frac{\textrm{tSupply}}{\textrm{rSupply}(0) - \sum\limits_{i=0}^{n-1}\textrm{rFee}(i)} \cdot \textrm{rOwned[user]}　　 　
\end{align}
$$

By combining equation (4) and (8), one can prove that denominator will not go to 0. 
$$
\begin{align}
\mathrm{balanceOf[user]}(n) = \frac{\textrm{tSupply}}{\prod\limits_{i=1}^{n} \left(1-\frac{\textrm{tFee}(n-i)}{\textrm{tSupply}} \right) \cdot \textrm{rSupply}(0)} \cdot \textrm{rOwned[user]}
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

Interestingly, this equation looks similar to equation (7).  
However, this method quickly becomes an issue after another transaction: 
$$
\begin{align*}
\mathrm {balanceOf[user]''} &=  \frac{\text{tSupply}}{\text{tSupply} - \mathrm{fee''}} \cdot \mathrm{balanceOf[user] '} \\
&= \frac{\text{tSupply}}{\text{tSupply} - \mathrm{fee''}} \cdot \frac{\text{tSupply}}{\text{tSupply} - \mathrm{fee'}} \cdot \mathrm{balanceOf[user] }
\end{align*}
$$ 

By introducing t-space and r-space, calculating balance becomes simpler with formula (8). 

## One issue with RFI contracts

RFI contracts has one fundamental issue as to re-including users into staking. The conversion rate between t-values and r-values is calculated using equation (5). There, it is excluding non-stakers out of equation in order to fully reward the stakers. However, when a user gets re-included into staking, the rate updates, thereby changing all stakers balance. Using this feature, this token can be used to rug-pull rewards from stakers by re-including a whale. 

Note: By default, all users are included in staking. Re-including means to exclude from staking then include after certain period of time. 

## Expanding the contract

Some projects expanded the contract to not only reward native tokens but also reward other tokens such as such as BNB, BTCB, CAKE, etc. This is done by taking some percentage of the reward, then convert it to other token using the router contract. 
However, this can lead to price drop since the native token will be sold every transaction. 

## About the author

Username: macroblock or REGO350
GitHub: https://github.com/REGO350
Contact: rego350xwb02@gmail.com 

