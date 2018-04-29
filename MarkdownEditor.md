System Contract Part #1 - Block Producer Rewards 
--

In eos network, [**eosio.system**](https://github.com/EOSIO/eos/tree/DAWN-2018-04-27-ALPHA/contracts/eosio.system) contract enables users to 1) stake tokens, and then vote on producers (or worker proposals), 2) proxy their voting influence to other users, 3) register producers, 4) claim producer rewards, 5) delegate bandwidth and push other necessary actions to keep blockchain system running.

In this series, we will go through this contract and talk about what is going on inside this contract and how to use it. 


From this article, questions on **Who is getting rewards**, **How much is the rewards**, and **How to claim rewards** will be discussed successively. 

**All codes present are based on commit of [9aa57f9](https://github.com/EOSIO/eos/tree/DAWN-2018-04-27-ALPHA)*

TL;DR:
--
* **Block producers get rewards of block production & votes proportion;**
* **Approx. 1.6 tokens issued on every second;**
* **For each producer, rewards claim can be approved at most once per 24 hours.**



Who Is Getting Rewards?
--
**21 active Block Producers (BP) and 100 candidates will be determined from voting. BPs are able to claim rewards from both the number of blocks they have produced as well as their votes proportion, i.e. Fixed Rewards & Dynamic Rewards respectively.**

BPs have to initiate rewards claim from eos system, some related system level parameters are defined in `global_state_singleton` as follows:

*@Global States*

|parameter name                 | description|  
|-------------------------------|:----------------------|
|`payment_per_block`            |Number of tokens assigned for one block production|
|`payment_to_eos_bucket`        |Number of tokens assigned to bucket per second *(bucket is used for dynamic rewards claim)*| 
|`first_block_time_in_cycle`    |Mark the start of a cycle |                      
|`blocks_per_cycle`             |Block count of a complete cycle|                       
|`last_bucket_fill_time`        |Last bucket fill time, usually be upon a new cycle   |                    
|`eos_bucket`                   |Bucket tokens balance, to be transferred to producers as dynamic rewards |

![image](https://github.com/oldcold/preview/blob/master/distribution.png)

How Much Will Producers Get?
--
**Amount of rewards is related to global parameters(e.g. inflation rate) and BP contribution. Based on the 5% inflation rate, approx. 1.6 tokens are issued system wise.**

Producer table includes information of all registered producers, which are widely used in system contract. Some columns related to producer rewards are listed below.

*@Producer Table*

| Column | Description |
|--------|-------------|
| **`owner`** |Producer account name|
|`total_votes`|Number of votes the producer has got| 
|`prefs`|Eosio parameters of the producer |
|`per_block_payments`|Fixed rewards available|
|`last_rewards_claim`|Last claim time, used to prevent from claiming too frequent|
|`last_produced_block_time`|Last time produced|


### Per Second Payment Calculation
**The amount of all rewards giving away is calculated using `system_contract::payment_per_block`**, determined by the product of system parameter `max_inflation_rate` (by default is 5%) and median of `percent_of_max_inflation_rate` from 21 active producers.


![equation](https://latex.codecogs.com/gif.latex?%24%24annual%5C_rate%20%3D%20%7B%7Bmax%5C_inflation%5C_rate%5Ctimes%20percent%5C_of%5C_max%5C_inflation%5C_rate%5Bmedian%5D%20%7D%5Cover%2010000%7D%24%24)


![equation](https://latex.codecogs.com/gif.latex?%24%24continuous%5C_rate%20%3D%20log1p%28annual%5C_rate%29%24%24)


![equation](https://latex.codecogs.com/gif.latex?%24%24per%5C_second%5C_payment%20%3D%20%7B2%5Ctimes%20%7B%7Bcontinuous%5C_rate%5Ctimes%20token%5C_supply.amount%20%7D%5Cover%20blocks%5C_per%5C_year%7D%7D%24%24)
	 


Currently there is no constraints on BPs' parameter `percent_of_max_inflation_rate`. If we assume it to be 100%, based on the 5% inflation rate from code and continuous rate formula, **maximum 1.5898 eos tokens are issued per second** (might be reduced if `max_inflation_rate` is modified in official release). This amount will be split into 2 "almost equal" parts for Fixed & Dynamic Rewards.



### Update Producer States From Election

**After an election, some of parameters related to rewards are being set, which are to be used in rewards calculation.**

1. The amount of token calculated in the previous section will be added into `parameters.payment_per_block` for Fixed Rewards and `parameters.payment_to_eos_bucket` for Dynamic Rewards.
	```cpp
    void system_contract::update_elected_producers(time cycle_time) {
      ...
      global_state_singleton gs( _self, _self );
      auto parameters = gs.exists() ? gs.get() :    get_default_parameters();
      ...
      auto half_of_percentage =       parameters.percent_of_max_inflation_rate / 2;
      auto other_half_of_percentage = parameters.percent_of_max_inflation_rate - half_of_percentage;
      parameters.payment_per_block = payment_per_block(half_of_percentage);
      parameters.payment_to_eos_bucket = payment_per_block(other_half_of_percentage);
      parameters.blocks_per_cycle = blocks_per_producer * schedule.producers.size();
      ...
    }
    
    ```
    
2. Corresponding tokens are issued to `eosio` account for further giveaway.

    ```cpp
    	...
      	auto issue_quantity = parameters.blocks_per_cycle * (parameters.payment_per_block + parameters.payment_to_eos_bucket);
      	INLINE_ACTION_SENDER(eosio::token, issue)( N(eosio.token), {{N(eosio),N(active)}},
                                                 {N(eosio), issue_quantity, std::string("producer pay")} );

      	set_blockchain_parameters( parameters );
      	gs.set( parameters, _self );
    }
    ```      

### Init New Cycle 
**Rewards rules are set upon every new cycle with the action of `system_contract::onblock`, inside this action, some global states and producer table will be updated accordingly.**

1. Update cycle

	```cpp
	void system_contract::onblock(const block_header& header) {
   // update parameters if it's a new cycle
   update_cycle(header.timestamp);
	...
	}
	
	bool system_contract::update_cycle(time block_time) {
		...
   
   		static const uint32_t slots_per_cycle = parameters.blocks_per_cycle;
   		const uint32_t time_slots = block_time - parameters.first_block_time_in_cycle;
   		if (time_slots >= slots_per_cycle) {
      		time beginning_of_cycle = block_time - (time_slots % slots_per_cycle);
      		update_elected_producers(beginning_of_cycle);
		...
	}
	   
	```
		
	
2. Modify producer table with **Fixed Rewards** that is to be claimed.

	```cpp
	void system_contract::onblock(const block_header& header) {
		...
   		//            const system_token_type block_payment = parameters.payment_per_block;
   		const asset block_payment = parameters.payment_per_block;
   		auto prod = producers_tbl.find(producer);
   		if ( prod != producers_tbl.end() ) {
      		producers_tbl.modify( prod, 0, [&](auto& p) {
            		p.per_block_payments += block_payment;
            		p.last_produced_block_time = header.timestamp;
         		});
   	}
	```

3. Fill `eos_bucket` for **Dynamic Rewards**.

	```cpp
	void system_contract::onblock(const block_header& header) {
		...
   		const uint32_t num_of_payments = header.timestamp - parameters.last_bucket_fill_time;
   		//            const system_token_type to_eos_bucket = num_of_payments * parameters.payment_to_eos_bucket;
   		const asset to_eos_bucket = num_of_payments * parameters.payment_to_eos_bucket;
   		parameters.last_bucket_fill_time = header.timestamp;
   		parameters.eos_bucket += to_eos_bucket;
   		gs.set( parameters, _self );
	}
	```

		
How To Claim Rewards
--

**BPs can claim rewards periodically (at most once a day), by pushing `claimrewards` actions. Fixed & Dynamic Rewards are calculated & transferred from `eosio` to the claimer.**

### BP Validation
1. Check whether the **account name is the same with the push sender**, which means producers cannot claim rewards on behalf of others.
2. Check whether the **push sender is among the producer list**, and being active (activity check might be removed in the coming versions).
3. Check whether producer is claiming too frequent, **no more than once a day**.

```cpp	
void system_contract::claimrewards(const account_name& owner) {
   	require_auth(owner);
   		eosio_assert(current_sender() == account_name(), "claimrewards can not be part of a deferred transaction");
   	producers_table producers_tbl( _self, _self );
   	auto prod = producers_tbl.find(owner);
   	eosio_assert(prod != producers_tbl.end(), "account name is not in producer list");
   	eosio_assert(prod->active(), "producer is not active"); // QUESTION: Why do we want to prevent inactive producers from claiming their earned rewards?
   	if( prod->last_rewards_claim > 0 ) {
      	eosio_assert(now() >= prod->last_rewards_claim + seconds_per_day, "already claimed rewards within a day");
   	}
   	...
}
```

### Fixed Rewards Calculation
This amount is set at every beginning of a new cycle mentioned above. 

```cpp
void system_contract::claimrewards(const account_name& owner) {
...   
	//            system_token_type rewards = prod->per_block_payments;
	eosio::asset rewards = prod->per_block_payments;
...
}
```

### Votes Calculation
**Dynamic rewards are calculated based on votes proportion, the number total of votes is obtained by iterating the producer table.**

1. Define a pointer index to the end of the producer table. This calculation will loop from newer registered producer to older.

	```cpp
	void system_contract::claimrewards(const account_name& owner) {
		...   
   		auto idx = producers_tbl.template get_index<N(prototalvote)>();
   		auto itr = --idx.end();     
   		...                         
	```
2. Loop from the tail to head, sum active producers' votes up into `total_producer_votes `. Which is all valid votes across the network. `num_of_payed_producers = 121`, defined in the system contract, consists of 21 BPs and 100 candidates.

	```cpp	
	void system_contract::claimrewards(const account_name& owner) {
	...   
   		bool is_among_payed_producers = false;                
   		uint128_t total_producer_votes = 0;
   		uint32_t n = 0;
   		while( n < num_of_payed_producers ) {                  
      		if( !is_among_payed_producers ) {            
         		if( itr->owner == owner )   
            		is_among_payed_producers = true;
      		}
      		if( itr->active() ) {
         		total_producer_votes += itr->total_votes;
         		++n;
      		}
      		if( itr == idx.begin() ) {
         		break;
      		}
      		--itr;
   		}

	...
	}
	```

### Dynamic Rewards Calculation
1. Get global states parameters.
2. Calculate **Dynamic Rewards** by  
![equation](https://latex.codecogs.com/gif.latex?%24%24gs.eos%5C_bucket.amount%5Ctimes%20%7Bproducer%5C_votes%20%5Cover%20total%5C_producer%5C_votes%7D%24%24)
3. Add dynamic rewards to `rewards`, this amount will be the final one transferred to the BP.
4. Deduct this dynamic rewards from the global `gs.eos_bucket`, the remaining amount will be claimed by others.

```cpp
   ...
   if (is_among_payed_producers && total_producer_votes > 0) {
      global_state_singleton gs( _self, _self ); 
      if( gs.exists() ) {
         auto parameters = gs.get();
         //                  auto share_of_eos_bucket = system_token_type( static_cast<uint64_t>( (prod->total_votes * parameters.eos_bucket.quantity) / total_producer_votes ) ); // This will be improved in the future when total_votes becomes a double type.
         auto share_of_eos_bucket = eosio::asset( static_cast<int64_t>( (prod->total_votes * parameters.eos_bucket.amount) / total_producer_votes ) );
         rewards += share_of_eos_bucket;
         parameters.eos_bucket -= share_of_eos_bucket;
         gs.set( parameters, _self );
      }
   }
   ...
}   
```

### Transfer Rewards To The Producer
1. Update producer table, set `last_reward_claim` to the current time, and reset block production rewards `per_block_payments` to be 0.

	```cpp
	void system_contract::claimrewards(const account_name& owner) {
   		...
      		//            eosio_assert( rewards > system_token_type(), "no rewards available to claim" );
   		eosio_assert( rewards > asset(0, S(4,EOS)), "no rewards available to claim" );

   		producers_tbl.modify( prod, 0, [&](auto& p) {
       		p.last_rewards_claim = now();
      		p.per_block_payments.amount = 0;
      	});
		...
	}
	```

2. Push an inline action, transferring corresponding amount from `eosio` account to the push sending producer.

	```cpp
	void system_contract::claimrewards(const account_name& owner) {
   		...

   		INLINE_ACTION_SENDER(eosio::token, transfer)( N(eosio.token), {N(eosio),N(active)}, { N(eosio), owner, rewards, std::string("producer claiming rewards") } );
	}
	```


For the time being rewards are distributed from a super user `eosio`, which means additional tokens will be created and assigned to the same account who is able to issue them upon the whole network launch.



Conclusion
--

1. Block producers get **Fixed Rewards** from blocks they are to produce, and **Dynamic Rewards** from proportion of votes they possess.

2. **Payment per block** might fluctuate based on BPs' parameters. Based on current configuration, **1.5898** eos tokens will become "visible" every second, which likely to be reduced upon release.

3. Producers have to **initiate** rewards claim from the system, no producer is able to claim more than **once a day**.

4. Some imperfect implementations (lack of constraints, etc) from the current code, we assume this is a stopgap for easy test and look forward to an improvement in the coming versions. 

*In the following articles, we are going to talk about some detailed implementation about* **voting process***, including producer registration, producer voting and proxy related stuff.*

**Stay tuned with [Eosio.SG](http://eosio.sg/): [Telegram](https://t.me/eosiosg), [Medium](https://medium.com/@eosiosg).**


