EOS系统合约 —— 投票
--

In eos network, [**eosio.system**](https://github.com/EOSIO/eos/tree/slim/contracts/eosio.system) contract enable users to 1) stake tokens, and then vote on producers (or worker proposals), 2) proxy their voting influence to other users, 3) register producers, 4) claim producer rewards, 5) delegate resources (net, cpu & ram) and push other necessary actions to keep blockchain system running.
在EOS网络中，区块链的系统合约允许用户 1）为区块生产者投票（或为提案投票）； 2）选择投票代理人； 3）注册成为区块生产者； 4）领取系统奖励； 5）委托计算、带宽等资源给其他用户等各种支持EOS网络正常运行的系统功能。
In this series, we will go through What is the contract, What are included in the contract and How the contract can be used.
在这个系列中，我们会详细介绍系统合约的内部运行机制，以及如何使用系统合约。
In this article, we will be discussing detailed flows/steps on **Producer Registration**, **Token Staking**, **Voting on BP** , and **Changing/Withdrawing Vote** successively. 
在这片文章中，我们会深入EOS源码，详细解析**注册生产者**、**购买股权**、**为生产者投票**以及**收回股权**等功能是如何实现的。

*All codes present are based on commit of [44e7d3e](https://github.com/EOSIO/eos/commit/44e7d3ef2503d7bc45afc18f04f0289ed26cfdd7)*
*本文中所有引用的代码来源于此次提交[44e7d3e](https://github.com/EOSIO/eos/commit/44e7d3ef2503d7bc45afc18f04f0289ed26cfdd7)*
TL;DR:
--
* **Token holders have to stake with their tokens on net and cpu for voting**
* **Token的持有者需要首先将使用Token购买股权才能进行投票**
* **On voting, all staked assets will convert to `x` amount of weighted votes, which can be used to vote up to 30 producers and each selected producer will get `x` amount of votes repectively**
* **在投票过程中，用户所有已购买的股权会转变为一定数量的选票，然后所有用户所选择的生产者（至多30个）都会增加这个数量的票数。**
* **Refunding process takes up to 3 days to reflect the unstaked tokens in available token balance**
* **股权赎回Tokens的过程需要等待3天**
* **Newer votes possess higher voting weights**
* **新产生的票数拥有更大的效力**

![voting](https://github.com/oldcold/preview/blob/master/voting.png)

Producer Registration
注册生产者
--
**Accounts should register themselves as producer first before they can be voted. This process is done by pushing a `system_contract::regproducer` action.**
**Token持有者只能为已经注册的生产者进行投票，注册生产者可以通过发送`regproducer`消息实现。**

* The core logic code below is to insert or replace producers' configurations (i.e. public key & parameters) into `producerinfo` table.
* 这段代码的核心逻辑是把生产者的配置（公钥，系统参数等）写入`producerinfo`表中，如果生产者曾经注册过，就更新之前的记录。
```cpp
void system_contract::regproducer( const account_name producer, 
                    const eosio::public_key& producer_key, const std::string& url ) { 
                    //, const eosio_parameters& prefs ) {
    ...

    if ( prod != _producers.end() ) {
        if( producer_key != prod->producer_key ) {
             _producers.modify( prod, producer, [&]( producer_info& info ){
                info.producer_key = producer_key;
            });
        }
    } else {
        _producers.emplace( producer, [&]( producer_info& info ){
            info.owner       = producer;
            info.total_votes = 0;
            info.producer_key =  producer_key;
        });
    }
}
```
**This part of code is under rapid development, we will keep updating it if significant changes are found.* 
**这部分代码仍处于快速开发的过程中，如果未来有大幅改动，我们会及时更新。*


    
Token Staking
购置股权
--
**Token holders can only vote after they have staked their tokens on net and cpu. Staking process is done by pushing a `system_contract::delegatebw` action. Inside `delegatebw` action, voter's tokens are staked and cannot be transferred until refunded.**
**Token持有者需要首先在带宽和计算资源上购置股权，这个过程是通过发送`delegatebw`的实现的。购置股权的Token不能用来交易，直到用户主动申请赎回。

1. If a user has not staked before, insert a record for this account in the table `deltable`. If a user has staked, add newly amount to the existing amount.
1. 如果用户没有购置过股权，在 `deltable`表中插入一条用户购置的股权信息。如果用户之前购置过股权，就在它原有的数额上增加新购置的部分。
2. Set resource limits for stake receiver. Transfer corresponding amount as stake to a public account `eosio`.
2. 设置股权接受者可以支配的资源限制，然后把购置的股权转移到`eosio`这个公共账户。
   ```cpp
    void system_contract::delegatebw( account_name from, account_name receiver,
                                     asset stake_net_quantity, 
                                     asset stake_cpu_quantity )                  
           {
              require_auth( from );
              ...

              set_resource_limits( tot_itr->owner, tot_itr->ram_bytes, 
                                tot_itr->net_weight.amount, tot_itr->cpu_weight.amount );

              if( N(eosio) != from) {
                 INLINE_ACTION_SENDER(eosio::token, transfer)( N(eosio.token), {from,N(active)}, 
                { 
                    from, N(eosio), asset(total_stake), std::string("stake bandwidth") 
                } );
          }
    ```
3. Update voter's staked amount.
3. 更新用户购置股权的金额。
    * Find the voter from `voters` table, if not exist, insert a new record of this voter.
    * 从`voters`表中找到投票用户的信息，如果用户此前从未参与投票，就在表中插入一列新的记录。
    * Add newly delegated stake into the voter's `staked` attribute.
    * 在用户的`staked`属性增加新购置的股权额度。
    * Call `voteproducer` action to update vote results. This means if the push sender has voted before, on new `delegatebw` action, votes will be updated for last voting producers (or lasting voting proxy).
    * 调用`voteproducer`消息，更新投票结果。也就是说，如果用户之前进行过投票，在这一次购置股权的时候，用户上次所有投票的生产者的票数都会根据当前用户的全部股权额度进行更新。
    ```cpp
    ...
          print( "voters \n" );
          auto from_voter = _voters.find(from);
          if( from_voter == _voters.end() ) {
            print( " create voter \n" );
             from_voter = _voters.emplace( from, [&]( auto& v ) {
                v.owner  = from;
                v.staked = uint64_t(total_stake);
            print( "    vote weight: ", v.last_vote_weight, "\n" );
            });
          } else {
            _voters.modify( from_voter, 0, [&]( auto& v ) {
                v.staked += uint64_t(total_stake);
                print( "    vote weight: ", v.last_vote_weight, "\n" );
             });
          }

          print( "voteproducer\n" );
          if( from_voter->producers.size() || from_voter->proxy ) {
            voteproducer( from, from_voter->proxy, from_voter->producers );
          }
       } // delegatebw
    ```

****Note that user can also delegate net & cpu to other accounts, making resource transfer to be possible. We will talk about user resources in depth in the upcoming blog.***
****用户购置的带宽和计算股权，可以把资源的使用权转交给其他用户，也就是说可以调用`delegatebw`来实现资源的配置。我们会在下一篇文章里深入讲解用户的资源分配话题。***



Vote On Producer / Proxy
为生产者/代理投票
--
**Stake holders (token holders who have their tokens staked) can vote for producers (or proxy, who will vote on behalf of push sender), all stakes will convert to weighted x votes and then add up to 30 producers by x votes.**
**股权的持有者，购置过带宽和计算资源股权的用户，可以为生产者（或为代理）投票。用户配置的全部股权会转变为加权后的票数x，用户所票选的生产者（至多30个）都会增加x票。
#### Vote producer
#### 为生产者投票
**Leaving `proxy` arguments to be empty*
**`proxy`参数置空*
1. Validation: 
1. 条件验证：
    * Producers to be vote must be given in order;
    * 票选的生产者需要按照一定的顺序排列（以节省查找时间）；
    * Producers to be vote must be registered;
    * 票选的生产者必须已经注册过；
    * Producers to be vote must be active. 
    * 票选的生产者必须是活跃状态。
2. Calculate current vote weight based on the following formula:
2. 根据下面的公式，计算当前的票数权重：
      
      ![equation](https://latex.codecogs.com/gif.latex?%24%24%u52A0%u6743%u7968%u6570%20%3D%20%u80A1%u6743%u6570%u989D%5Ctimes%7B%7B%u5F53%u524D%u65F6%u95F4%u6233%uFF08%u79D2%uFF09%5Cover%201000000%5Ctimes%20604800%20%28%u4E00%u5468%u7684%u79D2%u6570%29%5Ctimes%2052%20%7D%7D%24%24) 
      
**The weight increasing could be treated as a linear growing with time within a short period.*
**票数的权重增加过程可以近似看作线性，新票数权重相当于一年前票数权重的2倍。*

If the voter is a proxy, `proxied_vote_weight` of the voter will also be updated. 
如果投票的用户本身是代理，用户的`proxied_vote_weight`参数也会相应进行更新。

3. Reduce `last_vote_weight` (if ever), and then add current vote weight. 
3. 如果用户曾经有过投票，先将它上次投出的票数从相应的生产者中减去，然后再增加本次的总票数。
    * Create a relation between voting producer and vote weight.
    * 创建一个生产者和票数的对应关系。
    * Deduct last voting weight from voting producers.
    * 在生产者的票数中减去上次投票的加权票数。
    * Add each voting producer's vote weight by the new weight.
    * 未本次票选的生产者增加新权重对应的票数。

    ```cpp
    void system_contract::voteproducer( const account_name voter_name, 
                    const account_name proxy, const std::vector<account_name>& producers ) {
        require_auth( voter_name );

        ...

        boost::container::flat_map<account_name, double> producer_deltas;
        for( const auto& p : voter->producers ) {
            producer_deltas[p] -= voter->last_vote_weight;
        }

        if( new_vote_weight >= 0 ) {
            for( const auto& p : producers ) {
                producer_deltas[p] += new_vote_weight;
            }
        }
        ...
    }    
    ```

4. Record voting results.
4. 记录投票结果。
    * Modify `voters` table, update vote weight & voting producers (or proxy) respectively.
    * 修改`voters`表。更新用户的投票权重和选择的生产者列表（或代理）。
    * Modify `producerinfo` table, update producer's votes.
    * 修改`producerinfo`表。更新所选生产者的票数。

    ```cpp
        ...
        _voters.modify( voter, 0, [&]( auto& av ) {
            print( "new_vote_weight: ", new_vote_weight, "\n" );
            av.last_vote_weight = new_vote_weight;
            av.producers = producers;
            av.proxy     = proxy;
            print( "    vote weight: ", av.last_vote_weight, "\n" );
          });

        for( const auto& pd : producer_deltas ) {
            auto pitr = _producers.find( pd.first );
            if( pitr != _producers.end() ) {
                _producers.modify( pitr, 0, [&]( auto& p ) {
                p.total_votes += pd.second;
                eosio_assert( p.total_votes >= 0, "something bad happened" );
                eosio_assert( p.active(), "producer is not active" );
                });
            }
        }
    }
    ```

#### Vote proxy
#### 为代理投票
**Leaving `producers` arguments to be empty*
**`producers`参数指控*
>An account marked as a proxy can vote with the weight of other accounts which have selected it as a proxy. Other accounts must refresh their voteproducer to update the proxy's weight.
如果一个用户被标记为代理，那么他可以使用选择他作代理的用户票数权重。其他用户必须调用`voteproducer`以更新代理的权重。
1. Validation: 
1. 条件验证：
    * Proxy to be vote must have registered to be a proxy by pushing action `system_contract::regproxy`.
    * 票选的代理必须通过调用`system_contract::regproxy`的方式注册代理。
    * Proxy and producers cannot be voted at the same time.
    * 不能同时向生产者和代理投票。
2. Calculate current vote weight, same as above.
2. 计算当前的投票权重，同上。
3. Update proxy's vote weight
3. 更新代理权重
    * Deduct last voting weight from the voting proxy.
    * 从票选的代理人中减去上次投票的权重。
    * Add each voting proxy's vote weight by the new amount.
    * 为票选的代理人增加新的投票权重。
    ```cpp
        ...
        if( voter->proxy != account_name() ) {
            auto old_proxy = _voters.find( voter->proxy );
            _voters.modify( old_proxy, 0, [&]( auto& vp ) {
                vp.proxied_vote_weight -= voter->last_vote_weight;
                print( "    vote weight: ", vp.last_vote_weight, "\n" );
            });
          }

          if( proxy != account_name() && new_vote_weight > 0 ) {
            auto new_proxy = _voters.find( voter->proxy );
             eosio_assert( new_proxy != _voters.end() && new_proxy->is_proxy, "invalid proxy specified" );
             _voters.modify( new_proxy, 0, [&]( auto& vp ) {
                vp.proxied_vote_weight += new_vote_weight;
                print( "    vote weight: ", vp.last_vote_weight, "\n" );
             });
          }
     ```



Changing/Withdrawing Vote
改变/撤回投票
--

![delegating](https://github.com/oldcold/preview/blob/master/undelegating.png)

#### Votes Change
####改变投票

Voters are able to change voted producers (or proxy) by **pushing `voteproducer` actions again**, details have been discussed in the previous section. 
投票者可以通过再次调用`voteproducer`来实现改变选票，原有选择的生产者票数会被撤回，新的票数会被加在新选择的生产者上。我们在上一个部分已经详细讲解，不再赘述。
#### Votes Withdraw (Unstake)
#### 撤回投票（赎回股权）

**Voters can withdraw their votes by pushing by pushing `system_contract::undelegatebw` actions with any amount that is no bigger than the net & cpu been staked & delegated. Undelegated stakes will be available for `system_contract::refund` after 3 days.**
**用户可以通过调用`sundelegatebw`方法来撤回股权，计算和带宽资源撤回的额度不能超过之前购置的额度。赎回的股权可以在3天后申请回收到Token余额。
1. Decrease refunding amount from voter's `staked` column of `voter` table.
1. 从`voter`表中的`staked`列中减去用户申请赎回的股权额度。
2. Update `totals_tbl` table and update resource limits for the account.
2. 更新`totals_tbl`表，并且设置用户可以支配的资源限度。
3. Create refund request.
3. 创建退回请求。
    * Update `refunds` table with unstaked amount
    * 更新`refunds`表，设置赎回额度。
    * If user undelegate many times within a short period of time, the last undelegating time will be recorded (this time will be used for calculating the available refunding time).
    * 如果用户在短时间内进行多次赎回股权的操作，最后一次赎回的时间被记录（这个时间是用来计算用户何时可以申请回收Token到余额）。
    ```cpp
       void system_contract::undelegatebw( account_name from, account_name receiver,
                                       asset unstake_net_quantity, asset unstake_cpu_quantity )
   {
        ...
        auto req = refunds_tbl.find( from );
          if ( req != refunds_tbl.end() ) {
            refunds_tbl.modify( req, 0, [&]( refund_request& r ) {
                r.amount += unstake_net_quantity + unstake_cpu_quantity;
                r.request_time = now();
            });
          } else {
            refunds_tbl.emplace( from, [&]( refund_request& r ) {
                r.owner = from;
                   r.amount = unstake_net_quantity + unstake_cpu_quantity;
                   r.request_time = now();
               });
         }
        ...
    ```
4. Create (or replace) a deferred `system_contract::refund` transaction & update voting results. 
4. 创建（或替换）一个`system_contract::refund`延迟交易，并且更新投票结果。
    * Push a deferred transaction.
    * 发起一个延迟交易。
    * `refund_delay = 3*24*3600`, i.e. 3 days.
    * `refund_delay = 3*24*3600`，延迟3天后执行。
    * Call `voteproducer` to deduct corresponding votes from voted producers.
    * 调用`voteproducer`， 从用户投票的生产者上减去相应的票数。
    ```cpp
        ...
        eosio::transaction out;
          out.actions.emplace_back( permission_level{ from, N(active) }, _self, N(refund), from );
          out.delay_sec = refund_delay;
          out.send( from, receiver );

          const auto& fromv = _voters.get( from );

          if( fromv.producers.size() || fromv.proxy ) {
            voteproducer( from, fromv.proxy, fromv.producers );
          }
       } // undelegatebw
    ```

Conclusion
结论
--
1. Token owner can only vote after they **staked** their tokens on net & cpu.
1. Token的持有者需要在带宽和计算资源上**购置股权**才可以投票。
2. During voting action, all stakes of the voter will convert into x weighted votes, and every voted producer (up to 30) is going to get **equivalent x weighted votes**.
2. 在投票过程中，用户的所有股权会变成加权的x票，每个用户所选择的生产者（至多30个）会**分别增加x票**。
3. **Newer votes count more than older votes**, the weight grows approximately linearly.
3. **新的票数具有更大的权重**，票数权重的增加是一个接近线性的过程。
4. Users can undelegate their stakes and have to wait up to **3 days** before they can re-allocate this amount of tokens.
4. 用户可以赎回股权，但是如果需要等待**3天**才能重新交易或使用这部分金额来购置其他资源。

*In the following article, we are going to talk about some detailed implementation about **user resources**, including delegate cpu & net, buy & sell ram, new account related stuff.*
*在接下来的文章里，我们会讨论有关**用户资源**的具体实现，包括用户如何购置和支配带宽和计算资源，如何买卖内存资源，以及如何收费等问题。
**Stay tuned with [eosio.sg](http://eosio.sg/): [Telegram](https://t.me/eosiosg), [Medium](https://medium.com/@eosiosg).**
    
