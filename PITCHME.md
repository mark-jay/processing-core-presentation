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

 - validates business logic - user not deactivated/channel enabled/etc
 - generates transaction number
 - validates limits(will be explained later)
 - calculatesTariffs(will be explained later)
 - blocksMoney(updates wallets)
 - persist

---

### Transaction API. Possible operations. Create. limits

configurable depending on user's group(~0.3 sec):

 - wallets can't go lower than 0(all except one or two)
 - some wallets can't do more than X transaction per Y period
 - some can't go higher than Z money
 - some can't cashin more than X1 per Y1 period
 - some can't spend more than X2 per Y2 period
 - since all wallets are fetched at this point we have to lock them

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
for each account in debAccounts {
  account.availableBalance += ?
  account.currentBalance += ?
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

---

### Transaction API. Scalability. Create

In a single transaction

 - <span style="color:blue">validates business logic - fast non blocking read only operations</span>
 - <span style="color:blue">generates transaction number - fast non blocking read only operations</span>
 - <span style="color:red">validates limits - wallets fetched so they are in hibernate session</span>
 - <span style="color:red">calculatesTariffs</span>
 - <span style="color:red">blocksMoney</span>
 - <span style="color:red">persist</span>

---

### Transaction API. Scalability. Complete

In a single transaction

 - <span style="color:blue">validate limits</span>
 - <span style="color:red">unblocks money</span>
 - <span style="color:red">lockAllAccounts - does 'select for update' for each wallet in alphabetical order</span>
 - <span style="color:red">executeTariffs</span>
 - <span style="color:red">persistAll</span>
 - <span style="color:red">setStatus - COMPLETED</span>

---

### The end of existing processing

---

# New processing idea

---

### New processing idea. Overview

principles:

1. split into 3 independent services: allpay, processing(limits+money movements), tariffCalculator
2. move all limit-related logic to be database constraints
3. avoid hibernate entity fetching. Only db-updates.
4. keep db-transactions as small as possible manually fix what needed

---

### New processing idea. Basic idea of interaction

create:
http://www.plantuml.com/plantuml/svg/fPDDQnH148Rl_IjUy924zAwP5Iy19O6eU0b1z-2rxAd9DjiVcwvgbqNslsS_LcS78iPuQuggvtrgcBeIfQ8r1fEoCeg_doboX-iG5hJ2AtgeP1PSn8iAABLmEKQ_VKCB9I6dFYSilSuWIbe5Dr-kFquDRtgtJ7D0ZTvZIiLtdNnTYNAynCyZm7IrO0itevGuM53CDQb5LtAqq6p1wjPc0C1eWzp3DwmbXS3QN9xwrfut5xPbSMSM-_9aLnwz7LRV3Afhyy-Vm55mDP1o2zsRzLiVhNrNicCHd-wFFP-IVCBmZtezTQY8mW-LHNkLlypLHKlA23vwRP3J8ViRX1K_A5J6-Jiq5oPIcVzg8u5FhZ09jozAcZmlQVAofF5uZ9ZB8KkUFUSxK0Y7gJuNXRMMX3otC5bdN9PU63jD4dJa0xUdcb5o2D_9pNUK_Rg2GChb38RIRz39TkIarpAzoV2tplFNszQr-R_xDCVtUVVXzkRpxQS-_MhpgZ4y0W00

complete:
http://www.plantuml.com/plantuml/svg/XP2nJWCn44HxVyL8D20HqKSA2bJGKUG7Lhwvd2MVVTbT4qM8V-UEI4cewDPhpvlnRCr5lOqvlEGyoGchPtneZJHBPR_6bwiKa-YfblVk4NMYodBOn3fEcSxl44frGjD-SDJ-HeuxEJG9RUh4QL0U6irXT8EvU3Di46lfauxiWbTaSIgcCm4-S4HWwR0uX713NiqvpuddZ1V4_rbsQQIkGryLb3WWbQWKurF7yp1l85WKcRZv_3fWXUM94xlh-YsPLptDbnSKObDbyLV9KY9H5-1HSgRVrv9FMCoRKV6PU7pu8yrfJ6x11nglNHjig2t_re1UKhvsifsdDkOV
