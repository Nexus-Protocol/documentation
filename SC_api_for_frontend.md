# Docs for frontend

## How do you get LP reward list?

We have 3 incentivized swap pairs:
1. PSI-UST
2. PSI-nLuna
3. PSI-nETH

User will have rewards only if he:
1. provide liquidity to a pair
2. stake LP tokens for that pair

To get current rewards for a staking LP tokens query:
```json
{
  "staker_info": {
    "staker": "<staker address here>",
    "time_seconds": <current_time_in_seconds>
  }
}
```
to a staking smart contract.
response:
```json
{
  "staker": "<staker address here>",
  "reward_index": "0.55257762667755047",
  "bond_amount": "8407624037",
  "pending_reward": "0"
}
```
`pending_reward` is what you are looking for.

Each pair have its own staking smart contract.

On Bombay I instantiated:
- PSI-UST pool address: `terra1vu36vl68csfpm5faut372gj3g7mwlzfexncp38`
- PSI-nLuna pool address: `terra1aqcp4mav0cj6cxa0jgs38c3xjv6n5h5x52xt3g`
- PSI-nETH pool address: `terra15laqpswauhhmteqa7dny4fgzl4e9mf0sqcd2az`

## How do you claim a reward for LP Pool?

Execute `{"withdraw": {}}` message on staking LP contract.

## How do you stake LP tokens?

Execute `send` on LP token contract (cw20) with `{"bond": {} }` and contract address of Staking contract.

## Is it possible to provide liquidity and stake returned LP tokens in one transaction?

Yes, it is possible, via "AutoStake" function of staking SC.
In order to do that you need to send `auto_stake` message to staking SC. But, before that you need to send `increase_allowance` to cw20 contract of 
token that you want to provide.
For example, if you want to provide liquidity for `<cw20_token> - UST` pair, you need to execute 2 transactions:

1. On cw20 contract:
```json
{
  "increase_allowance": {
    "amount": "<some_amount>",
    "spender": "<staking_contract>"
  }
}
```

2. On staking contract:
```json
{
  "auto_stake": {
    "assets": [
      {
        "amount": "<cw20_tokens_amount>",
        "info": {
          "token": {
            "contract_addr": "<cw20_token_addr>"
          }
        }
      },
      {
        "amount": "<uust_amount>",
        "info": {
          "native_token": {
            "denom": "uusd"
          }
        }
      }
    ],
    "slippage_tolerance": "<slippage_tolerance>"
  }
}
```
Also do not forget to send coins: `coins - "[{\"amount\":\"<uust_amount>\",\"denom\":\"uusd\"}]"`
(it is not needed if you provide liquidity for `cw20 - cw20` pair).

You need to give user ability to choose `slippage_tolerance` (default value should be: `0.01`).

## How do you stake Psi?

Execute `send` on Psi token contract (cw20) with `{"stake_voting_tokens": {} }` and contract address of Governance contract.

## How do you get the amount deposited on bAsset Vault?

Query `custody_bAsset` contract with message `{ "borrower": { "address": "<basse_vault_addr>" } }` ([bombay-10 contract](https://finder.terra.money/bombay-10/address/terra1ltnkx0mv7lf2rca9f8w740ashu93ujughy4s7p))
response:
```json
{
  "borrower": "terra1rf3u2cdu2dc7tx46rqrcgwhh9mm3dd6zgwgeuf",
  "balance": "80000005",
  "spendable": "0"
}
```

`spendable` is bAsset that locked in `custody_bAsset`, but not used as collateral for loans.
`balance` is bAsset that locked in `custody_bAsset`, and used as collateral for loans.

I design our `bAsset_vault`'s in a way that they always use bAsset's as collateral, so for us `spendable` should always be zero. But, still, better to sum both fields.

## How to get ratio between LP token and assets in swap pair?

First, query pair contract with: `{"pool": {}}`
For example for [bLuna-Luna](https://finder.terra.money/columbus-4/address/terra1jxazgm67et0ce260kvrpfv50acuushpjsz2y0p) pair you will get:
You will get response:
```json
{
  "assets": [
    {
      "info": {
        "token": {
          "contract_addr": "terra1kc87mu460fwkqte29rquh4hc20m54fxwtsx7gp"
        }
      },
      "amount": "1388270752207"
    },
    {
      "info": {
        "native_token": {
          "denom": "uluna"
        }
      },
      "amount": "1364738411148"
    }
  ],
  "total_share": "1031011589707"
}
```
`total_share` means `total_supply` for LP token.

So, for 1LP token you will get: 
1. 1388270752207 / 1031011589707 = 1.346513236191194 `bLuna`
1. 1364738411148 / 1031011589707 = 1.3236887196736953 `Luna`

## How to get APR for LP?

1. Get `Psi` price by querying Psi-UST pool `{"pool": {}}` and dividing UST amount to PSI amount. ([bombay contract](https://finder.terra.money/bombay-10/address/terra1cz3l6yz7cdgwvx6he3pl7cf0em98r782auwle7))
2. Query staking SC with `{"config":{}}` json. ([bombay contract](https://finder.terra.money/bombay-10/address/terra194slkwm88juafc6fk4hfm6c5ncz9mknv6syf8z))
You will get response:
```json
{
  "owner": "terra1m6w200dw3gwfp2drrmj7amdwm47lg5w82ne6we",
  "psi_token": "terra1y6lc6t8f9vst037e49mec02x6e9aflct0l9ret",
  "staking_token": "terra1c5rma2etwygne033sfygs6ht3axetg70mn4s2e",
  "distribution_schedule": [
    {
      "start_time": 1631264400,
      "end_time": 1662800400,
      "amount": "100000000000000"
    }
  ]
}
```
You need distribution schedules (it is array). So, between `start_time` and `end_time` contract will distribute `amount` of PSI tokens.
I will create schedule for 1 year.
This mean you can calculate APR by this logic:
```python
psi_price = UST_amount_in_pool / PSI_amount_in_pool;
# divide by million cause all tokens is multiplied by millions in Terra.
psi_distribution_per_second = 100000000000000 / (1662800400 - 1631264400) / 1_000_000;
seconds_per_year = 60 * 60 * 24 * 365;
pool_ust_value = UST_amount_in_pool * 2 / 1_000_000;
APR = psi_price * psi_distribution_per_second * seconds_per_year / pool_ust_value * 100;
```

## How to get APR for bAsset vault?

Our vault profit is based on Anchor Earn and on Achor Borrow incentivizing.

### Get Anchor Earn APY
Query Anchor Overseer contract with `{"epoch_state": {}}` ([columbus-4 contract](https://finder.terra.money/columbus-4/address/terra1tmnqgvg567ypvsvk6rwsga3srp7e3lg6u0elp8))
response:
```json
{
  "deposit_rate": "0.000000041734138975",
  "prev_aterra_supply": "37334996603902",
  "prev_exchange_rate": "1.101147455743651484",
  "prev_interest_buffer": "3216298709528",
  "last_executed_height": 5613759
}
```
you need `deposit_rate`, it is in blocks.
To calculate APY you need to multiply it by number of blocks per year, which is `4656810`.

So, Anchor Earn APY: 
```python
deposit_rate = 0.000000041734138975;
number_of_blocks_per_year = 4656810;
anchor_earn_apy = deposit_rate * number_of_blocks_per_year * 100;
```

### Get Anchor Borrow Net APR

`Net APR` equals `Distribution APR` minus `Borrow APR`.
`Borrow APR` is interest.

1. [Calculate Anchor Borrow Distribution APR](#get-Anchor-Borrow-Distribution-APR)
2. [Calculate Anchor Borrow Interest APR](#get-Anchor-Borrow-Interest-APR)
3. [Calculate Anchor Borrow Net APR](#get-Anchor-Borrow-Net-APR)

#### **get `Anchor Borrow Distribution APR`**

Query Anchor Market contract with `{"state":{}}` ([bombay-10 contract](https://finder.terra.money/bombay-10/address/terra15dwd5mj8v59wpj0wvt233mf5efdff808c5tkal))
response:
```json
{
  "total_liabilities": "682703198917970.293507449953151794",
  "total_reserves": "0.127014529524242008",
  "last_interest_updated": 4465672,
  "last_reward_updated": 4465672,
  "global_interest_index": "1.11961568198213286",
  "global_reward_index": "0.214293652144318205",
  "anc_emission_rate": "20381363.85157231012364762",
  "prev_aterra_supply": "1229126648629140",
  "prev_exchange_rate": "1.096911382122411961"
}
```
To Calculate `Distribution APR` for Borrow incentivizing you need to multiply `anc_emission_rate` by number of blocks per year, which is `4656810` and by ANC price (which you can get from ANC-UST pool) and divide all that by `total_liabilities` and multiply by 100.
So, `Distribution APR`:
```python
anc_emission_rate = 20381363.85157231012364762;
anc_price = 2.909;
number_of_blocks_per_year = 4656810;
total_liabilities = 682703198917970.293507449953151794;
anchor_borrow_distribution_apr = anc_emission_rate * anc_price * number_of_blocks_per_year / total_liabilities * 100;
```

#### **get `Anchor Borrow Interest APR`**

You need to query Anchor Interest Model contract ([bombay-10 contract](https://finder.terra.money/bombay-10/address/terra1m25aqupscdw2kw4tnq5ql6hexgr34mr76azh5x)).

But in order to do that you need several values:
- `total_liabilities` (query `{"state":{}}` to Anchor Market contract)
- `total_reserves` (query `{"state":{}}` to Anchor Market contract)
- `market_balance` (UST amount in Anchor Market contract)
You can do that by `Bank` query in `terra-js`:
```javascript
const coins = await lcd_client.bank.balance(balance_config.address);
```

Okey, now query
```json
{
  "borrow_rate": {
    "market_balance": "653128483861401",
    "total_liabilities": "682703198917970.293507449953151794",
    "total_reserves": "0.127014529524242008"
  }
}
```
to Anchor Interest Model.
response:
```json
{
  "rate": "0.000000047824728815"
}
```

Now you can get `borrow_interest_apr`:
```python
anchor_interest_model_rate = 0.000000047824728815;\
number_of_blocks_per_year = 4656810;\
anchor_borrow_interest_apr = anchor_interest_model_rate * number_of_blocks_per_year * 100;
```

#### **get `Anchor Borrow Net APR`**

`anchor_borrow_net_apr = anchor_borrow_distribution_apr - anchor_borrow_interest_apr`

### bAsset vault APR

bAsset vault doing:
- borrow on Anchor
- split loan (UST) to buffer for emergency case and to depositing part
- deposit to Anchor Earn

#### get loan amount

First, query Anchor Market contract with `{"borrower_info": {"borrower": "<basse_vault_addr>"}}` ([bombay-10 contract](https://finder.terra.money/bombay-10/address/terra15dwd5mj8v59wpj0wvt233mf5efdff808c5tkal))
response:
```json
{
  "borrower": "terra1rf3u2cdu2dc7tx46rqrcgwhh9mm3dd6zgwgeuf",
  "interest_index": "1.06903089004734451",
  "reward_index": "709.565305833305095689",
  "loan_amount": "1137907203",
  "pending_rewards": "9724285.159495362927922183"
}
```
`loan_amount` is what we need.

#### get bAsset deposited to Anchor

Now, we need to get amount of bAsset deposited, you can find how to do it [here](#How-do-you-get-the-amount-deposited-on-bAsset-Vault).

#### get bAsset valut buffer size

Also, bAsset do not deposit to Acnhor Earn 100% of loan, it have some UST buffer for emergency case. So, you need to query what is UST balance on bAsset contract.
You can do that by `Bank` query in `terra-js`:
```javascript
const coins = await lcd_client.bank.balance(balance_config.address);
```

#### get bAsset price

Query Oracle contract with `{"price": {"base": "<basset_token_address>", "quote": "uusd"}}` ([bombay-10 contract](https://finder.terra.money/bombay-10/address/terra1p4gg3p2ue6qy2qfuxtrmgv2ec3f4jmgqtazum8))

bLuna address for `bombay-10`: `terra1u0t35drzyy0mujj8rkdyzhe264uls4ug3wdp3x`

response: 
```json
{
  "rate": "29.875290811644296225",
  "last_updated_base": 1631226300,
  "last_updated_quote": 9999999999
}
```

#### get Nexus Protocol fees

Nexus protocol use some part of profit to inflate Community Pool and to reward Governance stakers.
We should substract those from total basset vault apr.

In order to get those parts you need to query Nexus Psi Distributor contract with `{"config": {}}` ([bombay-10 contract for bLuna](https://finder.terra.money/bombay-10/address/terra1fs6nrhk5y2kd0x6gmkdahykfvh2yg0r5fn07m3))
(it is different for each bAsset)

response:
```json
{
  "psi_token_addr": "terra1y6lc6t8f9vst037e49mec02x6e9aflct0l9ret",
  "governance_contract_addr": "terra18xqvjxx0250hu3q40kndjh7q7kax3xuy3ak3kx",
  "nasset_token_rewards_contract_addr": "terra1lm2e27w677rgl0xsp9jzz6e8kz0e9zjr0lesl9",
  "community_pool_contract_addr": "terra1wasnf4frc88htrugz95lr4ynl99j8g2x67k2hv",
  "basset_vault_strategy_contract_addr": "terra1g4p0xvmwywm2ec3xgyhmjgj542wydg2zk36yum",
  "manual_ltv": "0.58",
  "fee_rate": "0.5",
  "tax_rate": "0.25"
}
```

Also, you need `borrow_ltv_aim`, get it from query Nexus bAsset vault strategy contract with `{"config": {}}` ([bombay-10 contract for bLuna](https://finder.terra.money/bombay-10/address/terra1g4p0xvmwywm2ec3xgyhmjgj542wydg2zk36yum))
(it is different for each bAsset)

response:
```json
{
  "governance_contract": "terra18xqvjxx0250hu3q40kndjh7q7kax3xuy3ak3kx",
  "oracle_contract": "terra1p4gg3p2ue6qy2qfuxtrmgv2ec3f4jmgqtazum8",
  "basset_token": "terra1u0t35drzyy0mujj8rkdyzhe264uls4ug3wdp3x",
  "stable_denom": "uusd",
  "borrow_ltv_max": "0.85",
  "borrow_ltv_min": "0.75",
  "borrow_ltv_aim": "0.8",
  "basset_max_ltv": "0.6",
  "buffer_part": "0.018",
  "price_timeframe": 50
}
```

Okey, now you can calculate Nexus fees:
```python
nexus_protocol_fee = (borrow_ltv_aim - manual_ltv) * fee_rate;
```

#### **calculat basset vault APR**

Okey, now we have everything we need to calculate `bAsset vault APR`:
```python
ust_in_basset_contract_address = 25602312 / 1_000_000;\
basset_deposited = 80000005 / 1_000_000;\
basset_price = 29.875290811644296225;\
loan_amount = 1137907203 / 1_000_000;\
basset_vault_ltv = loan_amount / (basset_deposited * basset_price);\
basset_vault_lending_portion = (loan_amount - ust_in_basset_contract_address) / (basset_deposited * basset_price);\
\
anchor_borrow_distribution_apr = 40.44208563570984;\
anchor_borrow_interest_apr = 22.12;\
anchor_borrow_net_apr = anchor_borrow_distribution_apr - anchor_borrow_interest_apr;\
anchor_earn_apy = 19.434795572016975;\
\
nexus_basset_vault_apr_from_loan = basset_vault_ltv * anchor_borrow_net_apr / 100;\
nexus_basset_vault_apr_from_lending = basset_vault_lending_portion * anchor_earn_apy / 100;\
\
nexus_basset_vault_net_apr = (nexus_basset_vault_apr_from_loan + nexus_basset_vault_apr_from_lending) * 100;\
\
nexus_basset_vault_strategy_aim_ltv = 0.8;\
anchor_borrow_users_manual_ltv = 0.58;\
nexus_basset_vault_strategy_fee_rate = 0.5;\
nexus_protocol_fee = (nexus_basset_vault_strategy_aim_ltv - anchor_borrow_users_manual_ltv) * nexus_basset_vault_strategy_fee_rate;\
nexus_basset_vault_net_apr_after_fee = nexus_basset_vault_net_apr * (1 - nexus_protocol_fee);
```

## How to get PSI governance staking APR?

Query Nexus PSI token for balance for Nexus Governance contract with `{"balance":{"address":"<governance_contract_addr>"}}` ([bombay-10 psi token](https://finder.terra.money/bombay-10/address/terra1y6lc6t8f9vst037e49mec02x6e9aflct0l9ret))

response:
```json
{
  "balance": "50100000000"
}
```
(nexus governance contract address for bombay-10: `terra1ax26e34fefah96r6svjgfs74fhmx7ztgat7ff6`)

Query Nexus Governance contract with `{"state":{}}` ([bombay-10 gov contract](https://finder.terra.money/bombay-10/address/terra1ax26e34fefah96r6svjgfs74fhmx7ztgat7ff6))

response:
```json
{
  "poll_count": 0,
  "total_share": "50100000000",
  "total_deposit": "0"
}
```

```python
# from cw20 contract
governance_psi_balance = 50100000000;\
\
# from query `{"state": {}}` on Governance contract
governance_total_share = 50100000000;\
governance_total_deposit = 0;\
\
gov_total_staked = governance_psi_balance - governance_total_deposit;\
gov_share_index = gov_total_staked / governance_total_share;\
```

Save `gov_share_index` every X blocks.

Calculate `epoch_interest` & `passed_blocks`:
```python
number_of_blocks_per_year = 4656810;\
passed_blocks = now(block_height) - 7_days_ago(block_height);\
epoch_interest = now(gov_share_index) - 7_days_ago(gov_share_index);\
\
gov_staking_apr = epoch_interest * number_of_blocks_per_year / passed_blocks * 100;
```

## How to calculate Psi circulation supply

Query Psi token with `{"token_info": {}}`

response:
```json
{
  "name":"Nexus Governance Token",
  "symbol":"Psi",
  "decimals":6,
  "total_supply":"10000000000000000"
}
```

So, we have `total_supply`. But some tokens is not in curculation, so we need to substract:
1. tokens in vesting contracts
2. tokens in LP staking contracts
3. tokens in community pool
4. vesting tokens for Pylon Swap (how to calculate them?)
5. vesting tokens for Pylon Pool (how to calculate them?)
6. Nexus Team tokens to vote in DAO for first 1 year (3% from total supply, which is 300_000_000)
7. tokens in Airdrop SC

So, we need to query all those contracts for Psi balance ([how to query cw20 token balance](#How-to-get-PSI-governance-staking-APR))
and substract from `total_supply`.

## How to claim Airdrop

User should send `Claim` message to Airdrop contract address ([bombay-11 airdrop contract](https://finder.terra.money/bombay-11/address/terra190ays2r593pmgkmxgjva2fklxgl2fux34qh8qz)):
```json
{
"claim":
	{
	"stage": 22,
	"amount": "530522770",
	"proof": [
			"40a4de0240f091bca5f0d1560f6de90446cc7f97fb5a752c408bffb27fe4385a",
			"b8a2e6111a6467bf9c4de5a26066f2ac4e2c7d074418f39b3a7b033a36e1e234",
			"14d1206bb1a90abe81d87cc425830bb5dee97a79e4c26d68de39e6b0c8f9d5b5",
			"ba9b2b2526857282eb8634e50460738078887e602edb156663e7617df5a16602",
			"17e21165cabec7ca34a49a3b4b8a870cf5626cf109ffb08ad0818a2b5181d1f0",
			"4ae82cf6c4f76d418e49f85e69f99b23a9fd8ecf3cc39a0342f91ad56fe08694",
			"c90db902b65e0bb32345bd472858395ae80948f75513d4e0a195830972aff3c2",
			"453c69622712d342341d54c5c4aabeb367a1c5da1c2746cd3becfbfffeca4085",
			"abb9f206febdb065cb63f8c535020eb8321f28f4ff4dcc69bb3ee4efc9f2656c",
			"6f8a9615a4170b58e487ad658ef9637b86f3c8954b42a8854b516cb1f1735e1d",
			"24e8c4eb0c7c1bdee7d9b82a860055a89db9726f9dedf58a22957232076767c9",
			"86302ec88a79e10d3cebca5dab857fd9dfaf50b9984aa7fd806395535e7e0067",
			"6e60ac703f10d30171ffcdf4a1dc5e66fab5d1b820b060aed9d5b3927bc14d04"
		]
	}
}
```
