# token合约剖析

孙伟杰 31/03/2019

---

- 1 代码分析(15min)
- 2 命令行演示(15min)

---

## 1 代码分析

- 1.1 multi_index table 介绍
  - 1.1.1 eosio.token
  - 1.1.2 eosiozjubca
- 1.2 action 介绍
  - 1.2.1 token action概要
  - 1.2.2 主要action分析
    - create
    - issue
    - transfer

---

### 1.1 multi_index table 介绍

- 1.1.1 eosio.token
- 1.1.2 eosiozjubca

---

#### 1.1.1 eosio.token

from `contracts/eosio.token/eosio.token.hpp`

```c++
struct account {
   asset    balance;
   uint64_t primary_key()const { return balance.symbol.name(); }
};
struct currency_stats {
   asset          supply;
   asset          max_supply;
   account_name   issuer;
   uint64_t primary_key()const { return supply.symbol.name(); }
};

typedef eosio::multi_index<N(accounts), account> accounts;
typedef eosio::multi_index<N(stat), currency_stats> stats;
```

---

#### 1.1.2 eosiozjubca

协会token

```c++
struct [[eosio::table]] account {
   asset    balance;
   uint64_t primary_key()const { return balance.symbol.code().raw(); }
};
struct [[eosio::table]] currency_stats {
   asset    supply;
   asset    max_supply;
   name     issuer;
   asset    frozen_amount;
   asset    release_amount_pertime;
   uint64_t next_release_day;
   uint64_t released_times;            
   uint64_t primary_key()const { return supply.symbol.code().raw(); }
};

typedef eosio::multi_index< "accounts"_n, account > accounts;
typedef eosio::multi_index< "stat"_n, currency_stats > stats;
```

---

> *According to the ‘eosio::multi_index’ definitions, `code` is the name of the account which has write permission and the `scope` is the account where the data gets stored.*

---

### 1.2 action 介绍

- 1.2.1 token action 概要
- 1.2.2 主要action分析
  - create
  - issue
  - transfer

---

#### 1.2.1 token action 概要

- **create()**
- **issue()**
- retire()
- **transfer()**
- open()
- close()

---

#### 1.2.2 主要action分析

- create
- issue
- transfer

---

##### create

```c++
void token::create( name   issuer,
                    asset  maximum_supply)
{
    require_auth( _self );

    auto sym = maximum_supply.symbol;
    eosio_assert( sym.is_valid(), "invalid symbol name" );
    eosio_assert( maximum_supply.is_valid(), "invalid supply");
    eosio_assert( maximum_supply.amount > 0, "max-supply must be positive");

    stats statstable( _self, sym.code().raw() );
    auto existing = statstable.find( sym.code().raw() );
    eosio_assert( existing == statstable.end(), "token with symbol already exists" );

    statstable.emplace( _self, [&]( auto& s ) {
       s.supply.symbol           = maximum_supply.symbol;
       s.max_supply              = maximum_supply;
       s.issuer                  = issuer;

       s.frozen_amount           = maximum_supply * 90 / 100;    // freeze 90% tokens since foundation
       s.release_amount_pertime  = maximum_supply * 3 / 100;   // unfreeze 3% every release day
       s.released_times            = 0;
       s.next_release_day        = 1538236800;              // 2018/9/30 00:00:00
    });
}
```

---

##### issue - main process

```c++
void token::issue( name to, asset quantity, string memo )
{
    auto sym = quantity.symbol;
    eosio_assert( sym.is_valid(), "invalid symbol name" );
    eosio_assert( memo.size() <= 256, "memo has more than 256 bytes" );

    stats statstable( _self, sym.code().raw() );
    auto existing = statstable.find( sym.code().raw() );
    eosio_assert( existing != statstable.end(), "token with symbol does not exist, create token before issue" );
    const auto& st = *existing;

    require_auth( st.issuer );
    eosio_assert( quantity.is_valid(), "invalid quantity" );
    eosio_assert( quantity.amount > 0, "must issue positive quantity" );

    eosio_assert( quantity.symbol == st.supply.symbol, "symbol precision mismatch" );
    
    /**
     * unfreeze token
    **/
    
    eosio_assert( quantity.amount <= st.max_supply.amount - st.supply.amount - st.frozen_amount.amount, "quantity exceeds available supply");

    statstable.modify( st, same_payer, [&]( auto& s ) {
       s.supply += quantity;
    });

    add_balance( st.issuer, quantity, st.issuer );

    if( to != st.issuer ) {
      SEND_INLINE_ACTION( *this, transfer, { {st.issuer, "active"_n} },
                          { st.issuer, to, quantity, memo }
      );
    }  
}
```

---

##### issue - freeze token

```c++
// ZJUBCA is founded on 2018/3/30 00:00:00 Unix time stamp: 1522339200
// Release date is set on 3.30 and 9.30 
uint64_t first_half = 3600 * 24 * 184; // first half: 3.30~9.30
uint64_t second_half_c = 3600 * 24 * 181; // second half for common year: 9.30~3.30
uint64_t second_half_l = 3600 * 24 * 182; // second half for leap year: 9.30~3.30
// Freeze 90% token for 15 years, release 3% every release day.
// update release date
if(now() >= st.next_release_day && st.released_times <= 30){
   if(st.released_times < 30){
      if(st.released_times % 2 == 1){ // first half
         statstable.modify( st, same_payer, [&]( auto& s ) {
         s.released_times += 1;
         s.frozen_amount -= st.release_amount_pertime;
         s.next_release_day += first_half;
         });
      }
      else if(st.released_times % 8 == 2){ // second half for leap year
      statstable.modify( st, same_payer, [&]( auto& s ) {
         s.released_times += 1;
         s.frozen_amount -= st.release_amount_pertime;
         s.next_release_day += second_half_l;
         });
      }
      else { // second half for common year
      statstable.modify( st, same_payer, [&]( auto& s ) {
         s.released_times += 1;
         s.frozen_amount -= st.release_amount_pertime;
         s.next_release_day += second_half_c;
         });
      }
   }
   else{
      statstable.modify( st, same_payer, [&]( auto& s ) {
      s.next_release_day = -1;
      });
   }
}
```

---

##### transfer

```c++
void token::transfer( name    from,
                      name    to,
                      asset   quantity,
                      string  memo )
{
    eosio_assert( from != to, "cannot transfer to self" );
    require_auth( from );
    eosio_assert( is_account( to ), "to account does not exist");
    auto sym = quantity.symbol.code();
    stats statstable( _self, sym.raw() );
    const auto& st = statstable.get( sym.raw() );

    require_recipient( from );
    require_recipient( to );

    eosio_assert( quantity.is_valid(), "invalid quantity" );
    eosio_assert( quantity.amount > 0, "must transfer positive quantity" );
    eosio_assert( quantity.symbol == st.supply.symbol, "symbol precision mismatch" );
    eosio_assert( memo.size() <= 256, "memo has more than 256 bytes" );

    //auto payer = has_auth( to ) ? to : from;
    
    sub_balance( from, quantity, st.issuer );
    add_balance( to, quantity, st.issuer );
}
```

---

## 2 命令行演示

- 2.1 deploy contract
- 2.2 create token
- 2.2 issue token
- 2.3 transfer token

---

end