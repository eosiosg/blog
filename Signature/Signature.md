#EOS Signature source code analysis

Signiture is the core part for every blockchain system. In the **EOS network**, there are several kinds of signiture, **transaction_signiture**, **block\_producer_signature**, **multi_signiture** and so on. In this article, we would like to introduce these signatures successively except **multi_signiture** from the source code.

⚠️**Warning**:  Before reading this article, make sure you have a clear mind and follow the source code step by step. Because you have a long tedious journey to walk.

*All the code is base on eos [slim branch](https://github.com/EOSIO/eos/tree/slim).*


## Trasaction signature 
In the EOS system, **controller** will wrap the transactions into the block, and one piece of **transaction** is constructed by **one or more actions**. Sign the transaction means you must sign every action occured in this transaction. The transaction will be **failed** and **will not write on the blockchain** if **any one action authorized failed**.

When a node join into the testnet. You should need to use **cleos** command. If you want to push some transactions such as transfer, regproducer, voteproducer, you should add the parameter -p (permission) to sign this transaction. Then the block producer will push these transactions to block, later need to be sign by this block producer. 

###Whole flow of transaction signature in function
![avatar](https://github.com/eosiosg/blog/blob/master/Signature/eos_signature.png)

Before you do this experiment, make sure you have start nodeos and keosd process.
Our team have open source
[one-click-boot script](https://github.com/eosiosg/testnet/tree/master/scripts)

```
cleos push action eosio.token transfer '["voter", "voter1", "100.0000 EOS","m"]' -p voter
cleos push action eosio regproducer '{"producer":"producer", "producer_key":"454f53355064556156723438395641384b356a525570736a524d35316847454445584e366a797043424e3535516e4264656f5a4577", "prefs":{"max_storage_size":10485760, "percent_of_max_inflation_rate": 0, "storage_reserve_ratio": 1000, "base_per_transaction_net_usage":0, "base_per_transaction_cpu_usage": 0, "base_per_action_cpu_usage":0, "base_setcode_cpu_usage":0, "per_signature_cpu_usage":0, "per_lock_net_usage":0, "context_free_discount_cpu_usage_num":0, "context_free_discount_cpu_usage_den":0, "max_transaction_cpu_usage":0, "max_transaction_net_usage":0, "max_block_cpu_usage":0, "target_block_cpu_usage_pct":0, "max_block_net_usage":0, "target_block_net_usage_pct":0, "max_transaction_lifetime":0, "max_transaction_exec_time":0, "max_authority_depth":0, "max_inline_depth":0, "max_inline_action_size":0, "max_generated_transaction_count":0, "max_transaction_delay":0 }}' -p producer
```

**These command tested in [DAWN-2018-04-27-ALPHA](https://github.com/EOSIO/eos/tree/DAWN-2018-04-27-ALPHA)*

The **-p** parameter means sign this transaction. Then we need to dive into the source code in [cleos main](https://github.com/EOSIO/eos/blob/slim/programs/cleos/main.cpp). 
There are some conventions you need to know in **cleos/main.cpp**. Nearly all subcommand functions have two key functions **send_actions** and **create_action**. Such as tranfer, regproducer, delegatebw, undelegatebw, voteproducer and so on.

For example, register producer subcommand.

```cpp
struct register_producer_subcommand {
  ...
         auto regprod_var = regproducer_variant(producer_str, producer_key, url );
         // create_action function and send_actions function
         send_actions({create_action({permission_level{producer_str,config::active_name}}, config::system_account_name, N(regproducer), regprod_var)});
}
```
	
###create_action
1. Create an action wrapping with action struct, including **account**, **action_name**, **vector of authorizations**, data.
2. One action have several authorizations, that means maybe there are **one or more than one** actors need to sign this action.

```cpp
chain::action create_action(const vector<permission_level>& authorization, const account_name& code, const action_name& act, const fc::variant& args) {
   auto arg = fc::mutable_variant_object()
      ("code", code)
      ("action", act)
      ("args", args);
	
   auto result = call(json_to_bin_func, arg);
   wlog("result=${r}",("r",result));
   // return a struct action with authorization vector
   return chain::action{authorization, code, act, result.get_object()["binargs"].as<bytes>()};
}
```
	
###send_actions
Send_action**s** rather than send\_action means **every action needs to be authorized**

```cpp
// send action with packed_transactions
void send_actions(std::vector<chain::action>&& actions, int32_t extra_kcpu = 1000, packed_transaction::compression_type compression = packed_transaction::none ) {
   auto result = push_actions( move(actions), extra_kcpu, compression);
	...
}
```
**We just list part of code, please refer to source code.*

###push_actions
1. The **vector of actions** are set in **transaction**
2. **compression** means this transaction need to **zip** or not before put into blockchain

```cpp
fc::variant push_actions(std::vector<chain::action>&& actions, int32_t extra_kcpu, packed_transaction::compression_type compression = packed_transaction::none ) {
   signed_transaction trx;
   trx.actions = std::forward<decltype(actions)>(actions);

   return push_transaction(trx, extra_kcpu, compression);
}
```

###push_transaction
1. The **push_transaction** is the **key function** of **action authorization**
2. first **determine\_required\_keys** get the **public_keys**
3. second **sign_transaction** with the required_key get before, we will disscuss it later

```cpp
fc::variant push_transaction( signed_transaction& trx, int32_t extra_kcpu = 1000, packed_transaction::compression_type compression = packed_transaction::none ) {
   ...
   // get required_keys to sign the transaction
   auto required_keys = determine_required_keys(trx);
   // mainly for multi-signature, but maybe didn't realize this part
   size_t num_keys = required_keys.is_array() ? required_keys.get_array().size() : 1;
	
   trx.max_kcpu_usage = (tx_max_cpu_usage + 1023)/1024;
   trx.max_net_usage_words = (tx_max_net_usage + 7)/8;
	
   if (!tx_skip_sign) {
   		// sign transaction
      sign_transaction(trx, required_keys);
   }
	
   if (!tx_dont_broadcast) {
      return call(push_txn_func, packed_transaction(trx, compression));
   } else {
      return fc::variant(trx);
   }
}
```
	
###determine\_required_keys
1. Call wallet API to fetch all unlocked(available) public keys, 
2. use authorization_manager to verify the require keys

```cpp
fc::variant determine_required_keys(const signed_transaction& trx) {
   const auto& public_keys = call(wallet_url, wallet_public_keys);
   auto get_arg = fc::mutable_variant_object
           ("transaction", (transaction)trx)
           ("available_keys", public_keys);
   
   const auto& required_keys = call(get_required_keys, get_arg);
   return required_keys["required_keys"];
}
```
If you want to sign a transaction, make sure you have keosd deamon which hold the wallet\_api plugin and wallet\_plugin. First to call wallet api to get all unlock public keys, and then to call the **chain_api** to verify the authorization from blockchain database.


###get\_required_keys
At the **determine\_required_key** function, it will call chain\_api, the origin **get\_required\_key** function is in the chain_plugin.cpp. 

1. The following function mainly to filter the legal key set from the **candidate_keys** that get from wallet.  
2. **make\_auth\_checker** will introduce later
3. Maybe there are **several actions** for each transaction, **the checker need to verify every action legality**.
4. The action design ideas come from [React Flux](https://github.com/facebook/flux/tree/master/examples/flux-concepts). Every action has a name and can dispatch with one or more handler. After dispatch hanlder the **action state** will be changed. 
5. Every action needs to be **authorized** and verified by checker.
6. If one action in transaction **miss the authorization**, the **whole transaction** will be **discarded**.

```cpp	
   flat_set<public_key_type> authorization_manager::get_required_keys( const transaction& trx,
                                                                       const flat_set<public_key_type>& candidate_keys,
                                                                       fc::microseconds provided_delay
                                                                     )const
   {
      auto checker = make_auth_checker( [&](const permission_level& p){ return get_permission(p).auth; },
                                        _control.get_global_properties().configuration.max_authority_depth,
                                        candidate_keys,
                                        {},
                                        provided_delay,
                                        _noop_checktime
                                      );

      for (const auto& act : trx.actions ) {
         for (const auto& declared_auth : act.authorization) {
            EOS_ASSERT( checker.satisfied(declared_auth), unsatisfied_authorization,
                        "transaction declares authority '${auth}', but does not have signatures for it.",
                        ("auth", declared_auth) );
         }
      }

      return checker.used_keys();
   }
```
	
Then how to get the legal key set? That's the key point which confuse me for a long time. Before this section you should have some **prerequisite** skill about C++ feature [**Operators Overloading**](https://www.tutorialspoint.com/cplusplus/cpp_overloading.htm).

Now, you can see the left part of **flow**. The second public **satisfied** funtion return and visitor. Actually It will call the **overloading ( )** function in weight\_tally\_visitor.

**There are 4 public satisfied override function and 1 private satisfied, make sure pick the right one*

1. The permission_level object which contain actor and permission (active, owner) needs to put in the cache for **checker performance**
2. **weight\_tally\_visitor** is to tally all weights, including (wait\_weight, key\_weight)
3. call the **operator ( ) overloading** function, return True if tally\_weight > 0.

```cpp
bool satisfied( const permission_level& permission, permission_cache_type* cached_perms = nullptr ) {
  permission_cache_type cached_permissions;
	
  if( cached_perms == nullptr )
     cached_perms = initialize_permission_cache( cached_permissions );
  weight_tally_visitor visitor(*this, *cached_perms, 0);
  // use the () overload function
  return ( visitor(permission_level_weight{permission, 1}) > 0 );
}
```

In this snippet, the author design a cache to put all the permission. It will increase the speed of find. 

1. Sometimes we are curious about how to convert the **permission** to **authority** object, because we didn't find the **permission\_to\_authority** function which is lambda function. Until I find the answer in the **misc_tests.cpp**
2. The checker will call **private satisfied** function adding the mapping<keys, account> to the permissions with sorted weights.

```cpp
uint32_t operator()(const permission_level_weight& permission) {
   auto status = authority_checker::permission_status_in_cache( cached_permissions, permission.permission );
   if( !status ) {
      if( recursion_depth < checker.recursion_depth_limit ) {
         bool r = false;
         typename permission_cache_type::iterator itr = cached_permissions.end();

         bool propagate_error = false;
         try {
            auto&& auth = checker.permission_to_authority( permission.permission );
            propagate_error = true;
            auto res = cached_permissions.emplace( permission.permission, being_evaluated );
            itr = res.first;
            r = checker.satisfied( std::forward<decltype(auth)>(auth), cached_permissions, recursion_depth + 1 );
         } catch( const permission_query_exception& ) {
         ...
}
```
For the checker.permission\_to\_authority lambda function. return an authority object which contain the relationship with **account** and **key**.

```cpp
auto GetNullAuthority = [](auto){abort(); return authority();};
```

###used_keys
The **\_used_keys** property is vector of bool to filter the candidate keys.

filter\_data\_by_marker Usage example:

	vector<char> data = {'A', 'B', 'C'};	
	vector<bool> markers = {true, false, true};
	auto markedData = FilterDataByMarker(data, markers, true);
	markedData contains {'A', 'C'}
	@endcode
	
```cpp
flat_set<public_key_type> used_keys() const {
  auto range = utilities::filter_data_by_marker(signing_keys, _used_keys, true);
  return {range.begin(), range.end()};
}
```

After these serval complicated function, we got the **required\_public_keys**! Later we need to use these keys to **sign the transactions**. Poping these stack function and back to push\_transaction. Let's dive into the **sign\_transaction** behavior! Please follow the **right part** of flow.

###sign\_transaction
1. We need to construct a **sign_args** which put into the **body of post request** for **wallet sign transaciton API**
2. call wallet API with suffix **/v1/wallet/sign_transaction**

```cpp
void sign_transaction(signed_transaction& trx, fc::variant& required_keys) {
   // TODO determine chain id
   fc::variants sign_args = {fc::variant(trx), required_keys, fc::variant(chain_id_type{})};
   // call the wallet api with suffix /v1/wallet/sign_transaction
   const auto& signed_trx = call(wallet_url, wallet_sign_trx, sign_args);
   trx = signed_trx.as<signed_transaction>();
}
```
	
The wallet_manager will fetch the **private\_key** base on the **public\_keys** we got in previous section.

```cpp
chain::signed_transaction
wallet_manager::sign_transaction(const chain::signed_transaction& txn, const flat_set<public_key_type>& keys, const chain::chain_id_type& id) {
   check_timeout();
   chain::signed_transaction stxn(txn);
   
   for (const auto& pk : keys) {
      bool found = false;
      for (const auto& i : wallets) {
         if (!i.second->is_locked()) {
            const auto& k = i.second->try_get_private_key(pk);
            if (k) {
            	//sign
               stxn.sign(*k, id);
               found = true;
               break; // inner for
            }
         }
      }
      if (!found) {
         EOS_THROW(chain::wallet_missing_pub_key_exception, "Public key not found in unlocked wallets ${k}", ("k", pk));
      }
   }
   return stxn;
}
```
This fragment is pretty clear and easy to understand. The **sign** action is performed in **transaction.cpp** that I will disscuss more in the next blog mainly focus on the controller.cpp to wrap the transactions to a whole block. That's the whole flow of **transaction signature**.

##Block signature
In this section, we will look more details for sign block. I won't introduce more about **controller.cpp** in this blog except the **sign block**. 

Before this section, you should know [**lambda function**](https://msdn.microsoft.com/en-us/library/dd293608.aspx) feature in C++. In the following function, the parameter is a call_back function which is called in **producer\_plugin.cpp**. 
###sign block
The **sign_block** function is called in **producer\_plugin.cpp** with a param lambda function. 

```cpp
void sign_block( const std::function<signature_type( const digest_type& )>& signer_callback ) {
  auto p = pending->_pending_block_state;
  p->sign( signer_callback );
  static_cast<signed_block_header&>(*p->block) = p->header;
} /// sign_block	
```

1. The **\_private_keys** is a public property map store in producer plugin, which I think will have some security issue.
2. **finalize_block** will generate the **action\_merkle\_root** and **transaction\_merkle\_root** and create block summary
3. **sign_block** will use the **private\_key\_itr->second** which is private_key to sign in **lamdba function**  
4. last step is commit block to the block chain, you can check more info in **controller.cpp**

```cpp
const auto& scheduled_producer = hbs->get_scheduled_producer( pending_block_timestamp );

auto private_key_itr = _private_keys.find( scheduled_producer.block_signing_key );
	
// for more detail need to check controller.cpp
chain.finalize_block();
	
// use the priavte_key list to sign the block
chain.sign_block( [&]( const digest_type& d ) { return private_key_itr->second.sign(d); } );
	
// commit block to chaindatabase
chain.commit_block();
```
	
	
## Conclusion

*  **Action or actions** must be signed correctly in **every transaction**.
* Analysis **sign_transaction** from the source code. You should be clear about the whole flow of signature, such as determine\_required\_keys, and sign transaction using private keys.
* You should **not** save your wallet private key in your node(if you own a node server). If you want to start a wallet in your node server, you should **disable** the wallet port or keep your wallet **locked**.
* Block signature is called in **producer_plugin.cpp** signed by scheduled_producers. 


*In the next article, we are going to talk more about **controller.cpp** some topic about block\_structure, block\_initialize, push\_transaction, finalize\_block, sign\_block, commit\_block.*
 
**Stay tune with** [**eosio.sg**](http://eosio.sg/) [**Telegram**](https://t.me/eosiosg)