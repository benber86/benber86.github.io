# Hybridizing Pangolin: Some thoughts on adding an order book to an AMM based exchange

<a href="#1-rationale">1. Rationale</a><br>
<a href="#2-hybrid-models">2. Hybrid models</a><br>
<a href="#21-centralized-solutions">&nbsp;&nbsp;&nbsp;&nbsp;2.1. Centralized solutions</a><br>
<a href="#22-relayers">&nbsp;&nbsp;&nbsp;&nbsp;2.2. Relayers</a><br>
<a href="#23-oneswap">&nbsp;&nbsp;&nbsp;&nbsp;2.3. Oneswap</a><br>
<a href="#24-uniswap-hybrid">&nbsp;&nbsp;&nbsp;&nbsp;2.4. Uniswap Hybrid</a><br>
<a href="#3-pangolin-hybrid">3. Pangolin Hybrid</a><br>
<a href="#31-modelling-price-mechanics-with-a-price-impact-function">&nbsp;&nbsp;&nbsp;&nbsp;3.1. Modelling price mechanics with a price impact function</a><br>
<a href="#32-closing-the-swap">&nbsp;&nbsp;&nbsp;&nbsp;3.2. Closing the swap</a><br>
<a href="#4-architecture">4. Architecture</a><br>
<a href="#41-storing-and-managing-the-order-book-on-chain">&nbsp;&nbsp;&nbsp;&nbsp;4.1. Storing and managing the order book on-chain</a><br>
<a href="#5-things-to-do">5. Things to do</a>

## 1. Rationale

Order books are a common tool used by exchanges, both centralized (CEX) and decentralized (DEX), to organize trades. Traders place buy and sell orders for assets, the orders are organized by price and matched through an order book. Order books are very efficient for liquid markets but will be very slow to match orders when liquidity is low. The liquidity issue was one of the drivers for the introduction of automatic market maker (AMM) models by DEXes such as Uniswap.<sup>1</sup> AMMs rely on liquidity reserves to guarantee liquidity on otherwise potentially illiquid markets. With AMMs, liquidity reserves are also used to set the traded asset prices - as opposed to the direct matching of buy and sell orders in an order book model.

One consequence of this difference is that placing limit orders is trivial in an order book based exchange: you simply specify the price at which you want to buy or sell and your order will be added to the order book until it is filled. This is not possible in an AMM DEX unless one automates the monitoring of price fluctuations and subsequent sell or buy orders.

Because Pangolin, like Uniswap, uses an AMM model, it is therefore currently not possible to place limit orders. The lack of such a feature is a strong disadvantage because, with the high volatility of crypto assets, limit orders are an essential tool to mitigate risk. The present document looks at ways such a feature could be implemented in addition to Pangolin's current AMM model.

## 2. Hybrid models

AMMs have largely supplanted the order book model among decentralized crypto exchanges. The order book model has several drawbacks that make it unsuited for the often illiquid markets of crypto as well as the high gas costs and relative slowness of on-chain transactions. Avalanche does solve the issue of gas fees and speed up to a certain extent and there have been rumors of an on-chain limit order book exchange coming to the network. However, the issue of limited liquidity will remain an issue especially for recently listed tokens.

On the other hand, order books also offer several advantages such as exposing more information about liquidity and market sentiment, increasing liquidity and allowing for limit orders.<sup>2</sup> For these reasons several hybrid models, drawing upon both AMMs and order books, have surfaced.

### 2.1. Centralized solutions

An easy possibility is to keep the order book on a centralized server. The centralized server would use the AMM as a price oracle, querying regularly for the current market price and filling the orders accordingly. 

### 2.2. Relayers

Relayer based order books rely on a network of (automated) node that monitoring an order book and market prices to fill orders appropriately. The main issue with the system is the added complexity of creating and incentivizing a network of relayers. 
0xProject's 0xmesh<sup>3</sup> relies on such a network of relayers that are tasked with both taking and filling orders. Relayers can also share orders with one another. 1inch  uses 0xmesh for its order book.
Pine.finance<sup>4</sup> uses a network of relayers to fill limit orders on Uniswap. Sushiswap<sup>5</sup> likewise implements a system based on relayers to handle its order book. As a gas saving measure, Sushi's order book system is partly deployed on the Kovan testnet. Traders place orders directly to the orderbook and relayers fill them partially or fully.

### 2.3. Oneswap

Oneswap<sup>6</sup> is the first to truly implement both an AMM model and an order book within the same hybrid solution. Their whitepaper remains vague on the technical specifics of the project, although it seems that the order book is handled on-chain by relying on a special data structure designed to optimize queries and writes. The project is open source and further detail could be obtained by looking at the code.

<img src="https://miro.medium.com/max/1660/1*WqfegbFFIiL4QhpWDHW6-Q.png"><br><sub><sup>Architecture of the Oneswap hybrid model</sup></sub>


### 2.4. Uniswap hybrid

In a single post on the Ethereum research forum<sup>7</sup> Theta Network's Jieyi Long outlines what a Uniswap hybrid could look like and offers an initial modelling attempt. It does not seem like this initial proposal generated any further research or implementations. However, it does outline an interesting algorithm for a `swap()` function based both on liquidity pools and order books. 

> The basic idea is straightforward. The DEX consists of a Uniswap-like liquidity pool, and an order-book which records the limit buy/sell orders. Whenever there is a token swap request, the limit orders will be taken first. Then, tokens in the liquidity pool get consumed, which moves the price, until it hits the next limit order. 

The author offers a formal description of the mechanism. However, that description seems to have limitations. The modelization rests on the assumption that the order book is a continuous set of prices. Whether this is adequate for crypto could be debated. For markets with low liquidity, one may expect significant price gaps between orders. From an implementation point of view, the price of an asset traded on an AMM can be a continuous variable, but an on-chain order book would have to be stored in a data structure like an array or a linked list. As replies to the original post also underline, having fixed intervals for orders is also more efficient and would save gas.


## 3. Pangolin hybrid

The draft below builds on the Uniswap hybrid modelization attempt but considers order prices to be a discrete variable separated by intervals. The prices can be stored in an index pointing to a FIFO queue where orders are stored. `{p0: [order queue at p0], p1: [order queue at p1], ..., pn: [order queue at pn]}`. This raises questions as to what data structure is appropriate, how to determine the interval, etc... which are left unaddressed in this section.

I'll start by describing how the `swap()` function introduced earlier can be used as a hybrid solution with an example.


### 3.1. Modelling price mechanics with a price impact function

We assume a dual token system in which only DAI (token <img src="https://render.githubusercontent.com/render/math?math=x">) and ETH (token <img src="https://render.githubusercontent.com/render/math?math=y">) are traded. There is one 50/50 liquidity pool, the reserve for DAI (<img src="https://render.githubusercontent.com/render/math?math=R_x">) is 100000 DAI and the reserve for ETH (<img src="https://render.githubusercontent.com/render/math?math=\R_y">) is 100 ETH.
The market price of ETH is 1000 DAI.
There are a number of sell orders for ETH at 1000, totaling 3 ETH.
Then there are a number of sell orders for ETH at 1010, totaling 2 ETH.

A trader places a market buy order for 10 ETH at 1000 DAI.

Sell orders are treated first, so the first 3 ETH are bought at 1000 DAI a piece with no impact on the liquidity pool.
There are now no sell orders until the price reaches 1010 DAI.
So the trader has to purchase from the pools until it reaches that point.
How much would the trader have to purchase?

The trader would add a number of DAI (<img src="https://render.githubusercontent.com/render/math?math=\Delta_x">) to <img src="https://render.githubusercontent.com/render/math?math=R_x"> and get a number of ETH (<img src="https://render.githubusercontent.com/render/math?math=\Delta_y">) from <img src="https://render.githubusercontent.com/render/math?math=R_y"> in exchange.

Because Pangolin is a constant function market maker, we know that <img src="https://render.githubusercontent.com/render/math?math=R_x \cdot R_y = (R_x %2B \Delta_x) \cdot (R_y - \Delta_y)">

From there it follows that <img src="https://render.githubusercontent.com/render/math?math=\Delta_x = \frac{R_x \cdot R_y}{R_y - \Delta_x}"> 

Now that we have expressed <img src="https://render.githubusercontent.com/render/math?math=\Delta_x"> as a function of <img src="https://render.githubusercontent.com/render/math?math=\Delta_y">, we can use the function's derivative to express the price of <img src="https://render.githubusercontent.com/render/math?math=\x"> as a function of <img src="https://render.githubusercontent.com/render/math?math=\Delta_y">: <img src="https://render.githubusercontent.com/render/math?math=\Phi(\Delta_y) = \frac{R_x \cdot R_y}{(R_y - \Delta_y)^{2}}"> 

So to find how much of  <img src="https://render.githubusercontent.com/render/math?math=y"> we need to buy to match a certain price  <img src="https://render.githubusercontent.com/render/math?math=p">, we simply have to solve <img src="https://render.githubusercontent.com/render/math?math=\Phi(\Delta_y) = p">

<img src="https://render.githubusercontent.com/render/math?math=\frac{R_x \cdot R_y}{(R_y - \Delta_y)^{2}} = p">

<img src="https://render.githubusercontent.com/render/math?math=\Delta_y = R_y - \sqrt{\frac{R_x \cdot R_y}{p}}">

Or in our example case: <img src="https://render.githubusercontent.com/render/math?math=\Delta_y = 100 - \sqrt{\frac{100000 \cdot 100}{1010}} \approx 0.4962809">

Let's round that to 0.5. So the next step in the trade would be to buy 0.5 ETH via AMM, bringing the price up to the next threshold of sell orders. At this point, there are 2 more ETH to be bought via the order book, with no effect on the liquidity pool. Once the sell orders have been filled, there is still 10 - (3 - 0.5 - 2) = 4.5 ETH to be purchased. The purchase will be filled through the AMM and bring the price to <img src="https://render.githubusercontent.com/render/math?math=\Phi(5) \approx 1108"> DAI.

### 3.2. Closing the swap

Proceeding iteratively is not very efficient, because it increases the number of transaction required to complete a trade. We assume that the cost of transactions for filling orders is null (paid for upon placement of the order by the person placing it and reimbursed as fees?). The only cost left to cover is that of the AMM swaps. However those can be aggregated in a single swap, we merely add the quantities needed to move the price as we fill orders and execute one single swap for that final quantity at the end. In pseudo-code:

```
amm_market_buy_total = 0
while (amount_to_buy > 0):
    sell_order_amount = sum(query_order_book[price])
    if (sell_order_amount >= amount_to_buy):
        fill_orders(amount_to_buy)
        break
    else:
        query_order_book.fill_orders(sell_order_amount)
        query_order_book.pop(price) // removes the orders we filled - this could be changed to return lowest value instead of lookup by price?
        amount_to_buy = amount_to_buy - sell_order_amount
        price = query_order_book.peek() // returns new lowest value
        amm_market_buy_amount = find_quantity_to_buy_to_reach_price(price)
        amm_market_buy_total = amm_market_buy_total + amm_market_buy_amount

amm_market_buy(amm_market_buy_total)
```

## 4. Architecture

### 4.1. Storing and managing the order book on-chain

Querying and writing new orders to the order book will require a specific data structure. Vitalik describes in a post on the Ethereum forums a method using a doubly linked list that could work for order books<sup>8</sup>. The post also suggests having hints sent by the client to cut down on computations needed to localize where the write should take place. This still leaves some question opens (what if someone sends the wrong hints on purpose, repeatedly?).

Need to look at the datastructure Oneswap implemented for comparison.

## 5. Things to do

- Get feedback on the basic idea
- Figure out the price interval question (minimum infinitesimal unit? how small? or proportional?)
- Rework the formula to account for fees
- Start looking into the order book contract from SUSHI and adapt for deployment on Fuji
- Create/delete test orders based on an unoptimized order book and run a few test sales
- Implement optimized data structure to store order book
- Implement hinting system
- Build first proof of concept


## Notes

[1] Hayden Adams. *Uniswap Whitepaper* (2018): https://hackmd.io/@477aQ9OrQTCbVR3fq1Qzxg/HJ9jLsfTz?type=view 

[2] Jamie E. Young. *On Equivalence of Automated Market Maker and Limit Order Book Systems* (October 2020): https://professorjey.com/assets/papers/AMM_Order_Book_Equivalence_DRAFT_2020_10_16.pdf 

[3] https://github.com/0xProject/0x-mesh

[4] https://github.com/pine-finance/pine-interface

[5] https://docs.sushi.com/products/limit-orders

[6] https://www.oneswap.net/whitepaper/OneSwap_WhitePaper_v1.0_en.pdf

[7] https://ethresear.ch/t/hybrid-of-order-book-and-amm-etherdelta-uniswap-for-slippage-reduction/7913/3

[8] https://ethresear.ch/t/doing-priority-queues-on-top-of-tries-more-efficiently-with-witnesses/94
