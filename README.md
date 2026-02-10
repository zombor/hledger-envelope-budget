# hledger envelope budgeting

YNAB style finances with `hledger`. Uses double entry accounting with envelope budgeting.

This document is heavily inspired by the [YNAB 4 Rules](https://www.youneedabudget.com/the-four-rules/). This method has kept my money in a good place for a long time:

1. Give Every Dollar a Job
2. Embrace Your True Expenses
3. Roll With The Punches
4. Age Your Money

## File Structure

Split your journal files into yearly files, with `current.journal` pointing at the current year:

```
<current-year>.journal
current.journal -> <current-year>.journal
```

Once you get enough years, make an `all.journal` file that includes every year so you can do historical analysis.

```
$ cat all.journal
include 2021.journal
include 2022.journal
```

## Account Structures

Because this is an adaptation of envelope budgeting there are some account prefix conventions that should be observed:

### Assets

1. `assets:cash` - This is any cash or cash equivilents you posess. I don't track actual cash once it leaves my bank account, so this is basically money in a bank account.
2. `assets:cash:<account>:checking` and `assets:cash:<account>:savings` - This is checking and savings for a single bank account. I only have one bank, so mine is `assets:cash:ally:checking`. This is where paychecks go when I get paid. More on this next.
3. `assets:cash:<account>:budget:checking|savings:<category>` - These are envelopes where spendable money goes. This is where _all_ spendable money goes. This means that your actual checking and savings accounts should have a zero balance in your ledger. When you add up the budget categories in your `budget:checking:*` accounts, it should add up to the amount in your physical checking account. This way you can never overspend your balance as long as you follow your envelopes. It is a bit deceiving, but it's good for tracking!
4. `assets:cash:<account>:budget:checking|savings:unallocated` - This is money that you don't have a job for yet. All money needs a category, even if it's not assigned yet.
5. `assets:cash:<account>:budget:checking|savings:pending:<credit-card>` - This is money that is set aside to pay a credit card. Whenever you buy something on a credit card, you'll transfer money from your budget envelope to this account. This ensures that you always have enough money to pay your credit card at the end of the month. It's scoped to checking or savings so that you can keep track of where to pay from. It's probably pretty rare to pay a credit card from a savings account, so I'd expect this to only be a checking sub-account. Doing this also lets you reconcile your `assets:cash:<account>:budget:checking` account with your actual checking balance. If you run `hledger bal <account>:budget --cleared`, this number should match your physical bank balance.

### Liabilities

These follow standard `hledger` conventions: `liability:<name>`, where name is a descriptor for the credit card, loan, etc.

### Expenses

These follow standard `hledger` conventions: `expenses:<name>`, where name is a descriptor for the expense type. I find it useful to match your budget category names.

## The Four Rules

So how do we use the 4 rules with `hledger`?

### Rule 1: Give Every Dollar A Job

This means when you receive income, every cent that you received should be assigned an account category.

For regular job income I record net received, not the gross. Taxes and such are "invisible" in this case. It also means any retirement transfers that are pre-tax do not show up from income. We'll cover that later.

I get paid on the 1st and 15th of the month.

```
2021-12-01 Paycheck
  assets:cash:ally:checking  $1000
  income:salary
```

Note that this violates one of the rules stated above: money never actually sits in a physical checking account, it must be assigned a job. So if you know what job all your money has immediately, send the money to those accounts. Otherwise, put it in the `unassigned` account:

```
2021-12-01 Budgeting
   assets:cash:ally:budget:checking:mortgage               $600.00
   assets:cash:ally:budget:checking:utilities              $100.00
   assets:cash:ally:budget:checking:groceries              $100.00
   assets:cash:ally:budget:checking:unallocated            $200.00
   assets:cash:ally:checking  = 0
```

Note the `balance assertion` at the end of this. This ensures that you moved all your money into budget categories.

To ensure you did this correctly, you can use `hledger bal -BEt assets:cash` to check. `assets:cash:ally:budget:checking` should equal the actual amount in your bank account at that time. `assets:cash:ally:checking` should be `0`.

Now that your money has a job, you can spend it!

```
2021-12-14 People's Gas
   expenses:utilities                 $96.04
   assets:cash:ally:budget:checking:utilities
```

Now when you check your balances with `hledger bal -BEt budget:utilities` you'll see that you have $3.97 left to spend. This amount will stay there forever until you move, add to it or spend the remaining. This transaction was pretty simple because usually you'll pay your utilities from your checking account. But what about paying with a credit card?

```
2021-11-27 Amazon
   expenses:groceries                    $3.41
   assets:cash:ally:budget:checking:groceries    $-3.41
   assets:cash:ally:budget:checking:pending:amazon-visa  $3.41
   liabilities:amazon-visa
```

This is a little more complicated, there's 4 postings! This is explained by:

1. Add to the expenses category
2. Subtract from the budget category
3. Add to the appropriate credit card pending account so we have money to pay
4. Subtract from the credit card because we owe more money

The `assets:cash:ally:budget:checking:pending` account is meant to hold money to pay credit cards. In this way, you can never overspend your credit card and not have enough money to pay it. So let's pay it now:

```
2021-12-30 Visa Payment
   assets:cash:ally:budget:checking:pending:amazon-visa  $-3.41
   liabilities:amazon-visa  = 0
```

Another `balance assertion` to make sure we're paying the full amount of the card.

### Rule 2: Embrace Your True Expenses

This is heavily related to `Rule 1`. The gist is that you want to find large expenses like car insurance, christmas gifts, capex for real estate, etc and put small amounts of money to them every month. So if you know that you have a $600 car insurance payment every 6 months, make sure you put $100 into that budget category every month.

### Rule 3: Roll With The Punches

This means when you overspend a category, don't fret! It's normal. Simply move money from another category that has an abundance to the negative one. If you have extra money in `unallocated`, this is the best place to start:

```
2021-11-28 Budget Transfer
   assets:cash:ally:budget:checking:spending-money:me  $43.17
   assets:cash:ally:budget:checking:unallocated
```

If you have no money in the unallocated category, it needs to come from another category (or more than one)

```
2021-11-28 Budget Transfer
   assets:cash:ally:budget:checking:spending-money:me   $43.17
   assets:cash:ally:budget:checking:groceries          $-20.00
   assets:cash:ally:budget:checking:fuel               $-23.17
```

### Rule 4: Age Your Money

The gist of this rule is that you want to have at least one month's expeses in reserve. Another way to put it is that the average age of money in your bank account should be 30 days old.

I haven't figured out how to calculate this in `hledger` yet, but I am pretty sure it's possible!

## More Example Transactions

Okay, so how to do other things?

### Mortgage/Loan Payments

1. Subtract from the budget category
2. Add interest expense (we pay money for the loan)
3. Add escrow expense (if there is one)
4. Add remaining amount to loan liability (we owe less to the bank)

```
2021-11-28 Mortgage
   assets:cash:ally:budget:mortgage  $-1,000.00
   expenses:interest:mortgage           $100.10
   expenses:escrow:mortgage             $225.00
   liabilities:mortgage
```

### Credit Card Cashback

We got paid from amex directly to our amex balance

1. Add the amount to our amex liability (we owe less)
2. Add the amount to our unallocated budget (or another one if you know where it could go) (we have more to spend)
3. Remove the amount from our pending amex payment (we don't have to pay amex as much)
4. Remove the amount from the income account where the payment came from

```
2021-11-26 AMEX | Dining Credit
   liabilities:amex                                     $10
   assets:cash:ally:budget:unallocated                  $10
   assets:cash:ally:budget:checking:pending:amex       $-10
   income:amex
```

### Pre-tax investments

For things like a 401k or HSA that comes out of income pre-tax by the employer, you can still record that as income (since it is: you would have received cash and then transfered the funds anyway). These don't involve budgets though.

```
2021-11-26 401k contribution
   assets:investments:401k  314.5 VTSAX @ $114.65
   income:401k
```

If you receive an employer match and want to record that, use two income lines:

```
2021-11-26 401k contribution
   assets:investments:401k  314.5 VTSAX @ $114.65
   income:401k:match        $-500.00
   income:401k
```

### After-tax investments

I budget money for after-tax investments, so it's basically a normal transaction except you trade USD for another asset minus fees.

```
2021/11/26 Coinbase
   assets:investments:coinbase                30.700514 ADA @ $1.56
   expenses:fees:coinbase                     $1.99
   assets:investments:coinbase                0.0175312 ETH @ $4,107.53
   expenses:fees:coinbase                     $2.99
   assets:investments:coinbase                0.00131575 BTC @ $54,729.24
   expenses:fees:coinbase                     $2.99
   expenses:fees:coinbase                     $0.12 ; Rounding
   assets:cash:ally:budget:checking:investing:crypto   $-200.00
```

### Credit Card Payments

1. Add the payment amount to your liability account (you paid the debt)
2. Subtract the payment amount from `assets:cash:<account>:budget:checking:pending:<card>` (you're sending money from your checking account to pay the debt)

```
2021-11-29 Amex Payment
   liabilities:amex                                $1,234.56 = 0
   assets:cash:ally:budget:checking:pending:amex  $-1,234.56 = 0
```
