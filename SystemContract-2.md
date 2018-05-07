System Contract Part #2 - Voting Process 
--

In eos network, [**eosio.system**](https://github.com/EOSIO/eos/tree/slim/contracts/eosio.system) contract enables users to 1) stake tokens, and then vote on producers (or worker proposals), 2) proxy their voting influence to other users, 3) register producers, 4) claim producer rewards, 5) delegate resources (net, cpu & ram) and push other necessary actions to keep blockchain system running.

In this series, we will go through this contract and talk about what is going on inside this contract and how to use it. 


In this article, we will be talking about detailed implementations on **Producer Registration**, **Token Staking**, **Voting on BP** , and **Votes Change & Withdraw** successively. 

*All codes present are based on commit of [44e7d3e](https://github.com/EOSIO/eos/commit/44e7d3ef2503d7bc45afc18f04f0289ed26cfdd7)*

TL;DR:
--
* **Token holders should stake with their tokens on net and cpu**
* **On voting, all staked tokens will convert to an identical amount of votes for up to 30 producers**
* **Refunding process (from staked status) takes up to 3 days to be available in token balance**
* **Newer votes possess higher voting weights**

![voting](https://github.com/oldcold/preview/blob/master/voting.png)

Producer Registration
--
**Every account should register to be a producer first before it can be voted. This process is done by pushing a `system_contract::regproducer` action.**


* The core logic code below is to insert or replace producers' configurations (i.e. public key & parameters) into `producerinfo` table.

```cpp
void system_contract::regproducer( const account_name producer, const eosio::public_key& producer_key, const std::string& url ) { //, const eosio_parameters& prefs ) {
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



    
Token Staking
--
**Token holders are eligible to vote if they have staked on net and cpu. Staking process is done by pushing a `system_contract::delegatebw` action. Inside `delegatebw` action, voter's tokens are staked and cannot be transferred until refunded.**

1. If a user has not staked before, insert a record for this account in the table `deltable`. If a user has staked, add newly amount to the existing amount.
2. Set resource limits for stake receiver (same with sender when voting). Transfer corresponding amount as stake to a public account `eosio`.

   ```cpp
    void system_contract::delegatebw( account_name from, account_name receiver,
                                     asset stake_net_quantity, 
                                     asset stake_cpu_quantity )                  
           {
              require_auth( from );
              ...

              set_resource_limits( tot_itr->owner, tot_itr->ram_bytes, tot_itr->net_weight.amount, tot_itr->cpu_weight.amount );

              if( N(eosio) != from) {
                 INLINE_ACTION_SENDER(eosio::token, transfer)( N(eosio.token), {from,N(active)}, 
                { 
                    from, N(eosio), asset(total_stake), std::string("stake bandwidth") 
                } );
          }
    ```
3. Update voter's staked amount.
    * Find the voter from `voters` table, if not exist, insert a new record of this voter.
    * Add newly delegated stake into the voter's `staked` attribute.
    * Call `voteproducer` action to update vote results. This means if the push sender has voted before, on new `delegatebw` action, votes will be updated for last voting producers (or lasting voting proxy).
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



Vote On Producer / Proxy
--
**Stake holders can vote producers (or proxy, who will vote producers on behalf of push sender), all stakes will convert to weighted n votes and then add up to 30 producers by n votes.**

#### Vote producer
**Leaving `proxy` arguments to be empty*
1. Validation: 
    * Producers to be vote must be given in order;
    * Producers to be vote must be registered;
    * Producers to be vote must be active. 
2. Calculate current vote weight based on the following formula:
      
      ![equation](https://latex.codecogs.com/png.latex?%5CLARGE%20%24%24votes%20%3D%20stakes%5Ctimes%7B2%5E%7B%7Bcurrent%5C_timestamp%5C_in%5C_second%5Cover%20604800%20%28seconds%5C_per%5C_week%29%5Ctimes%2052%20%7D%7D%20%5Csimeq%20ax%5Ctimes%20stakes%7D%24%24) 
      
**The weight increasing could be treated as a linear growing with time within a short period.*

If the voter is a proxy, `proxied_vote_weight` of the voter will also be updated. 

3. Reduce `last_vote_weight` (if ever), and then add current vote weight. 
    * Create a relation between voting producer and vote weight.
    * Deduct last voting weight from voting producers.
    * Add each voting producer's vote weight by the new weight.

    ```cpp
    void system_contract::voteproducer( const account_name voter_name, const account_name proxy, const std::vector<account_name>& producers ) {
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
    * Modify `voters` table, update vote weight & voting producers (or proxy) respectively.
    * Modify `producerinfo` table, update producer's votes.

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
**Leaving `producers` arguments to be empty*

>An account marked as a proxy can vote with the weight of other accounts which have selected it as a proxy. Other accounts must refresh their voteproducer to update the proxy's weight.

1. Validation: 
    * Proxy to be vote must have registered to be a proxy by pushing action `system_contract::regproxy`.
    * Proxy and producers cannot be voted at the same time.
2. Calculate current vote weight, same as above.
3. Update proxy's vote weight
    * Deduct last voting weight from the voting proxy.
    * Add each voting proxy's vote weight by the new amount.
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



Votes Change & Withdraw
--

![delegating](https://github.com/oldcold/preview/blob/master/undelegating.png)

#### Votes Change

Voters are able to change voted producers (or proxy) by **pushing `voteproducer` actions again**, details have been discussed in the previous section. 

#### Votes Withdraw (Unstake)

**Voters are able to withdraw their votes, by pushing `system_contract::undelegatebw` actions with any amount less or equal to the net & cpu staked they have delegated. Undelegated stakes will be available for `system_contract::refund` after 3 days.**

1. Decrease refunding amount from voter's `staked` column of `voter` table.
2. Update `totals_tbl` table and update resource limits for the account.
3. Create refund request.
    * Update `refunds` table with unstaked amount
    * If user undelegate many times within a short period of time, the last undelegating time will be recorded (this time will be used for calculating the available refunding time).
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
    * Push a deferred transaction.
    * `refund_delay = 3*24*3600`, i.e. 3 days.
    * Call `voteproducer` to deduct corresponding votes from voted producers.
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
--
1. Only tokens **staked** on net & cpu are available on voting.
2. During voting action, every voted producer (up to 30) is going to get **identical weighted votes**, whose amount is calculated based on staked amount.
3. **Newer votes count more than older votes**, the weights grows approximately linearly.
4. Users can undelegate their stakes and have to wait up to **3 days** before they can re-allocate this amount of tokens.

*In the following article, we are going to talk about some detailed implementation about **user resources**, including delegate cpu & net, buy & sell ram, new account, producer voting and proxy related stuff.*

**Stay tuned with [eosio.sg](http://eosio.sg/): [Telegram](https://t.me/eosiosg), [Medium](https://medium.com/@eosiosg).**
    
