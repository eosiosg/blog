#EOS签名源码分析

签名是每个区块链系统的核心内容. 在EOS网络中, 有许多种签名, **transaction_signature**, **block\_producer_signature**, **multi_signature** 等等. 在这篇文章中我们会从源码层面上陆续介绍**transaction_signature**, **block\_producer_signature**.

*所有源码是基于[9be8910](https://github.com/EOSIO/eos/tree/dawn-v4.0.0).*

##交易签名(Transaction signature)

在EOS系统中, **controller**对象会包装 **transactions**到一个区块中. 每一个**transaction** 会包含一个或多个**actions**. 为了成功对这比交易进行签名, 这里的**每一个action**都必须签名正确. 如果某一个签名错误, 会导致这比交易全部失败. 并不会写到链上. 在其他的情况下导致的失败, 如(hard fail, soft fail) 会写到区块链上.

在**cleos**命令中, 有一些命令需要签名操作, **-p**参数需要指明一个签名者, 对这笔交易签名. 不仅如此, 当一个区块**生产者(block producer)** 将**交易(transaction)** 打包成区块的时候, 也要对这个区块进行签名.

###签名的主要流程图
![avatar](https://github.com/eosiosg/blog/blob/master/Signature/eos_signature.png)

*以下将详细介绍图上的所有函数和流程*

下面举个例子方便理解, 在这之前, 希望你可以搭建起来nodeos环境. [scripts](https://github.com/eosiosg/scripts)

```
cleos push action eosio.token transfer '["voter", "voter1", "100.0000 EOS","m"]' -p voter
cleos push action eosio regproducer '{"producer":"producer", "producer_key": "EOS8dCVTUjdCv94GB7wr7vFTJri7iRYd7c4y6U8tHGaK3peMerUr5", "url":"http://eosio.sg"}' -p producer
```

上面两个命令的**-p**就是对这个**transaction**进行签名. 从这里开始我们要深入的去看下源码[cleos main](https://github.com/EOSIO/eos/blob/dawn-v4.0.0/programs/cleos/main.cpp). 首先需要了解下这个文件的**套路**. 几乎所有的**subcommand**函数都有两个核心函数组成 **create_action** 和 **send_actions**.

例如, 注册成为区块生产者(regproducer)

```cpp
struct register_producer_subcommand {
  ...
         auto regprod_var = regproducer_variant(producer_str, producer_key, url );
         // create_action function and send_actions function
         send_actions({create_action({permission_level{producer_str,config::active_name}}, config::system_account_name, N(regproducer), regprod_var)});
}
```

###create_action
1. 创建一个**action** 结构体包含着**用户(account)**, **用户名(account_name)**, **授权数组(vector of authorizations)** 和 **数据(data)**.
2. 一个操作有一个**授权数组(vector of authorizations)**. 说明会有至少一个actor需要对这个操作进行授权.

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
这里的send_action**s**, 并不是send\_action也从侧面证明了一笔**transaction**, 会有一个到多个的操作**(action)**需要签名.

```cpp
// send action with packed_transactions
void send_actions(std::vector<chain::action>&& actions, int32_t extra_kcpu = 1000, packed_transaction::compression_type compression = packed_transaction::none ) {
   auto result = push_actions( move(actions), extra_kcpu, compression);
	...
}
```

###push_actions
1. **actions**数组变成了**transaction**的一个属性.
2. **compression** 的意思是这比交易在打包到区块上是否需要压缩.

```cpp
fc::variant push_actions(std::vector<chain::action>&& actions, int32_t extra_kcpu, packed_transaction::compression_type compression = packed_transaction::none ) {
   signed_transaction trx;
   trx.actions = std::forward<decltype(actions)>(actions);

   return push_transaction(trx, extra_kcpu, compression);
}
```

###push_transaction

1. **push_transaction**函数是**action**签名中的核心函数
2. 首先, 要调用**determine\_required\_keys**去拿到所有的可用的 **public_keys**
3. 然后, 用上一步拿到的**public_keys**去钱包中映射出**private_key**对这个**action**进行签名. 这两个步骤会分别在后面进行讨论.

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

1. 调用钱包的API函数取得所有**解锁的(可用的)public keys**
2. 用 **authorization_manager** 从区块链的数据库中验证需要的keys

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

*确保 **keosd** 进程起来, 这个进程里会有 wallet\_api plugin and wallet\_plugin. *

###get\_required_keys
在**determine\_required_key**函数中, 会去调用 chain\_api, 原始的**get\_required\_key**是在 chain_plugin.cpp.

1. 下面的函数主要要从wallet中拿到的**候选keys** 过滤出合法的key集合.
2. **make\_auth\_checker** 会在后面的函数中介绍
3. 下面的循环就是要对一笔**交易(transaction)**每一个**action**是否合法进行检查.
4. **action**的设计理念主要来源于 [React Flux](https://github.com/facebook/flux/tree/master/examples/flux-concepts). 每一个action有个名字. 能够被一个或者多个hanlder进行处理, 在触发handler之后, action的状态会发生变化.
5. 如果**transaction** 中的一个**action** 授权失败, 整个交易全部失败


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


如何去拿到合法的key set? 这个问题也困扰了我很久. 在讨论这个问题之前, 需要了解下C++的一些语言特性, [**运算符重载**](https://www.tutorialspoint.com/cplusplus/cpp_overloading.htm).

现在, 切换到流程图的左侧. 

这里一共有5个重载(**override**)的**satisfied**函数. 4个公有, 1一个私有(后面会用到).

第**二个**公有函数 **satisfied** 返回 **visitor**. visitor是一个对象, 当后面是**( )**, 就是在调用**( )**运算符重载函数.

1. **permission_level**对象包含了actor和permission 两个属性, 为了提高系统的性能将permission放在了缓存中.
2. **weight\_tally\_visitor**是所有权重的统计, 包括(wait\_weight, key\_weight)
3. 调用 **( ) 运算符重载** , 如果所有的权重 > 0, 返回true.

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

1. 在 **checker.permission\_to\_authority( permission.permission )**中, permission\_to\_authority是一个**lambda函数**.

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

可以在**misc_tests.cpp**找到更详细的函数写法. 下面就是一个lambda函数, 具体的lambda函数写法要参照[**lambda function**](https://msdn.microsoft.com/en-us/library/dd293608.aspx), 下面的函数会返回一个包含**account** 和 **key** 映射的对象 **authority**

```cpp
auto GetNullAuthority = [](auto){abort(); return authority();};
```
具体的**authority**对象定义在 **eos/libraries/chain/include/eosio/chain/authority.hpp**中.

3. checker对象会调用 **private satisfied**函数. 这时会把**key**和 **account**的关系放入到 **permission** 对象中. 对permission进行遍历, 这里又用到了C++中的**观察者模式**, 在**permission.visit(visitor)**会调用第二个**( )运算符重载**函数, 改变**_used_keys**, 把key的权重增加到全部权重中.
4. 这个权重重要是利用在多人签名合约中. 在一个action中只有一个人需要签名的时候, 并不需要这么复杂.

```cpp
 template<typename AuthorityType>
 bool satisfied( const AuthorityType& authority, permission_cache_type& cached_permissions, uint16_t depth ) {
    // Save the current used keys; if we do not satisfy this authority, the newly used keys aren't actually used
    auto KeyReverter = fc::make_scoped_exit([this, keys = _used_keys] () mutable {
       _used_keys = keys;
    });

    // Sort key permissions and account permissions together into a single set of meta_permissions
    detail::meta_permission_set permissions;

    permissions.insert(authority.waits.begin(), authority.waits.end());
    permissions.insert(authority.keys.begin(), authority.keys.end());
    permissions.insert(authority.accounts.begin(), authority.accounts.end());

    // Check all permissions, from highest weight to lowest, seeing if provided authorization factors satisfies them or not
    weight_tally_visitor visitor(*this, cached_permissions, depth);
    for( const auto& permission : permissions )
       // If we've got enough weight, to satisfy the authority, return!
       if( permission.visit(visitor) >= authority.threshold ) {
          KeyReverter.cancel();
          return true;
       }
    return false;
 }
```

###used_keys
 **\_used_keys**是一个bool类型的数组, 用来过滤从钱包取出来的候选的key.这里的一个核心函数是FilterDataByMarker, 看了下面的例子简单易懂.

filter\_data\_by_marker :

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


在这一系列复杂的函数之后, 我们从钱包中过滤掉了签名需要的**public_keys**. 之后我们需要用这些key去签名一笔交易. 现在我们来看如何去签名一笔交易, 这部分比前一部分简单很多, 现在请转移到图的右半部分.

###sign\_transaction

1. 首先要构建一个放入post请求体**post request body** 中的参数 **sign_args**
2. 然后去调用wallet API **/v1/wallet/sign_transaction**

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

这里出现的另外一个重要的对象就是**wallet_manager**, 基于我们之前的**public\_keys** 取出相应的**private\_key** 用来签名一笔交易.

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

这一段非常的清楚比较容易理解. 这个小节最重要的内容就是如何去授权部分. authorization_manager, 作者把签名设计的极其复杂, 主要还是针对多人签名, 如果感兴趣的话, 可以去看下cleos中的multi-signature 源码和测试代码. 有助于理解这部分的代码.

##区块签名(Block signature)

在这一段来了解更多关于区块签名的细节, 这里只说**controller.cpp** 中的区块签名 **sign block**.

在这个小节之前, 需要知道一些C++的一些特性[**lambda function**](https://msdn.microsoft.com/en-us/library/dd293608.aspx). 

###sign block

在下面的函数中, 函数参数是一个回调函数, 在 **producer\_plugin.cpp**调用.

```cpp
void sign_block( const std::function<signature_type( const digest_type& )>& signer_callback ) {
  auto p = pending->_pending_block_state;
  p->sign( signer_callback );
  static_cast<signed_block_header&>(*p->block) = p->header;
} /// sign_block	
```


1. **\_private_keys**作为一个公有属性存在了**producer plugin**中.
2. **finalize_block**会最终生成**action\_merkle\_root** 和 **transaction\_merkle\_root**, 最后也会生成区块的总结.
3. 区块的签名会在 **lamdba function** 用到**private\_key\_itr->second** (代表private_key)去签名
4. 最后一步是提交区块在区块链上, 参照 **controller.cpp** 获取更多的信息.

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

## 总结

* 在一笔交易中的每一个**Action or actions** 都一定要签名正确, 否则这一比交易会被取消掉.
* 从源码层面上分析了交易的签名, 主体函数 **determine\_required\_keys**, 用**private keys**去签名一笔交易.
* 在一个node节点上不要存储自己的钱包**private_key**, 如果需要在自己的节点上开钱包就要禁用自己的钱包端口, 或者让自己的钱包在不用的时候上锁.
* 区块签名是区块生产者在打包完成区块之后, 对整个区块做的一个签名. 在**producer_plugin.cpp** 调用.


*在下一篇blog中, 我们会介绍更多关于**controller.cpp** 的内容, 比如区块的结构 (block\_structure), 区块初始化 (block\_initialize), 将交易打包成区块(push\_transaction), 区块的完成(finalize\_block), 更加详细的区块签名(sign\_block), 以及 区块的提交上链(commit\_block)*
 
**加入我们更多讨论** [**eosio.sg**](http://eosio.sg/) [**Telegram**](https://t.me/eosiosg)
