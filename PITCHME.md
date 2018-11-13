# Processing core

---

### Business problems
 - slow processing time(in terms of throughput) because existing solution is not scalable(1 tx/second - terrible)
 - limits not working correctly(we disabled them that's why we're having no problems right now)

---

### How wallet is stored

 - current balance - what user have
 - available balance - what user can spend

---

### Operations on a wallet. Blockings

 - block amount - 'wallet.availableBalance -= X'
 - unblock amount - 'wallet.availableBalance += X'

---

### Operations on a wallet. Transfer

transfer X money from wallet1 to wallet2:

 - wallet1.availableBalance -= X
 - wallet1.currentBalance -= X
 - wallet2.availableBalance += X
 - wallet2.currentBalance += X

---

### Transaction API. Possible operations

 - create - validates & blocks money
 - complete - validates again & sends money
 - decline - does nothing, returns everything back as it was before

---

### Transaction API. Possible workflows

- create -> complete
- create -> decline
- failed during 'create' operation

---

### Transaction API. Possible operations. Create operation overview

In a single transaction

 - validates business logic - user not deactivated/channel enabled
 - generates transaction number
 - validates limits(will be explained later)
 - calculatesTariffs(will be explained later)
 - blocksMoney(updates wallets)
 - persist

---

### Transaction API. Possible operations. Create. limits

limits validation
goes through all wallets and makes sure the following requirements are met(configurable):

 - wallets can't go lower than 0(all except one or two)
 - some wallets can't do more than X transaction per Y period
 - some wallets can't go higher than Z money
 - some wallets can't cashin more than X1 credit per Y1 period
 - some wallets can't spend more than X2 credit per Y2 period
 - since all wallets are fetched at this point we have to lock them
 - validateLimits(transaction); // ~0.3 sec as it goes through user's history

---

### Transaction API. Possible operations. Create. tariffs calculation

 - creates a lot of rows associated with the current transaction
 - each row is related to a specific tariff and represent a money movement

---

### Transaction API. Possible operations. Complete operation overview

In a single transaction:

 - validate limits
 - unblocks money
 - lockAllAccounts - does 'select for update' for each wallet in alphabetical order
 - executeTariffs(will be explained later)
 - persistAll
 - setStatus - COMPLETED

---

### Transaction API. Possible operations. Complete. Execute tariffs

all that was previously calculated in method 'calculateTariffs'
is now executed by generating a lot of 'TransactionHistory' rows
each such row represents an increase or decrease(debit or credit) on a wallet explaining why it was done so
```
for each account {
  account.availableBalance += X
  account.currentBalance += X
  persist(new TransactionHistory(...))
}
```

---

### Transaction API. Possible operations. Decline operation overview

In a single transaction:

 - unblocksMoney
 - setStatus - DECLINED
 - persists

---

### Transaction API. Possible operations. Link

https://gist.github.com/mark-jay/606af433376ca4d7ed25b10acd3bf91e

### The end
