#gracenode-wallet Module

Coin management module for gracenode framework.


### Requirements

In order for gracenode-wallet module to work properly, you need to add gracenode-mysql module to your application.

#### Before you start using gracenode-wallet

gracenode-wallet module uses mysql database to store validation data, you will need to create the required table for the module.

To create the required mysql table, you will need to execute the following SQL queries:

`gracenode-wallet/schema.sql`

If you need to execute the queries from Node.js application, you may do:

```
var gracenode = require('gracenode');
gracenode.setConfigPath('path/to/your/config/dir/');
gracenode.setConfigFiles(['yourConfig.json']);
gracenode.use('gracenode-mysql');
gracenode.use('gracenode-wallet');
gracenode.setup(function (error) {
	if (error) {
		return console.error(error);
	}
	gracenode.getModuleSchema('gracenode-wallet', function (error, sqlList) {
		if (error) {
			// hmm error
		}
		// execute the SQL queries in sqlList array here
	});
});
```

## How to include it in my project

To add this package as your gracenode module, add the following to your package.json:

```
"dependencies": {
	"gracenode": "",
	"gracenode-mysql": "",
	"gracenode-wallet": ""
}
```

To use this module in your application, add the following to your gracenode bootstrap code:

```
var gracenode = require('gracenode');
// this tells gracenode to load the module
// make sure you load gracenode-mysql module BEFORE gracenode-wallet module
gracenode.use('gracenode-mysql');
gracenode.use('gracenode-wallet');
```

To access the module:

```
// the prefix gracenode- will be removed automatically
gracenode.wallet
```

Configurations
```javascript
"modules": {
	"gracenode-wallet": {
        	"names": [an array of wallet names],
        	"sql": "mysql configuration name"
	}
}
```

#####API: *create*

<pre>
Wallet create(String walletName)
</pre>
> Returns an instance of Wallet class by a wallet name
>> The wallet name needs to be defined in the configuration file

##### Wallet class

> **getBalanceByUserId**
<pre>
void getBalanceByUserId(String uniqueUserId, Function callback)
</pre>
> Rerturns the current balance (paid and free separately) of a wallet in the callback as a second argument

> **add**
<pre>
void add(String uniqueReceiptHash, String uniqueUserId, Int price, Object values, Function onCallback<optional>, Function callback)
</pre>
> Adds "paid" and/or "free" to a wallet.
```
// this will add 100 paid and 30 free into the wallet "hc".
var hc = gracenode.wallet.create('hc');
hc.add(receipt, userId, { paid: 100, free: 30 }, handlOnCallback, finalCallback);
```

> **addPaid**
<pre>
void addPaid(String uniqueReceiptHash, String uniqueUserId, Int price, Int value, Function onCallback<optional>, Function callback)
</pre>
> Adds the value to a wallet as "paid"
>> "paid" represents that the user has paid real money

>> If onCallback is given: the function will be called BEFORE committing the "add" transaction, if an error occuries in onCallback, the transaction can be rolled back

> **addFree**
<pre>
void addFree(String uniqueReceiptHash, String uniqueUserId, Int value, Function onCallback<optional>, Function callback)
</pre>
> Adds the value to a wallet as "free"
>> "free" represents that the user has been given the value as free gift

>> If onCallback is given: the function will be called BEFORE committing the "add" transaction, if an error occuries in onCallback, the transaction can be rolled back

Example:
```javascript
// example code with iap module
gracenode.iap.validateApplePurchase(receipt, function (error, response) {
        if (error) {
                // handle error here
        }

        // check the validated state
        if (response.validateState === 'validated') {
                // Apple has validated the purchase

                var hc = gracenode.wallet.create('hc');
                hc.addPaid(receipt, userId, itemPrice, itemValue,

                        // this callback will be called BEFORE the commit of "addPaid"
                        function (continueCallback) {

                                // update iap status to mark the receipt as "handled"
                                gracenode.iap.updateStatus(receipt, 'handled', function (error) {
                                        if (error) {
                                                // error on updating the status to "handled"
                                                return continueCallback(error); // this will make "addPaid" to auto-rollback
                                        }

                                        // iap receipt status updated to "handled" now commit
                                        continueCallback();

                                })

                        },

                        // this callback is to finalize "addPaid" transaction
                        function (error) {
                                if (error) {
                                        // error on finalizing the transaction
                                }

                                // we are done!
                        }

                );

        }

});
```

> **spend**
<pre>
void spend(String uniqueUserId, Int value, String spendFor, Function onCallback, Function callback)
</pre>
> Spends value from a wallet if allowed
>> spendFor should represent what the user has spend the value for

>> If onCallback is given: the function will be called BEFORE committing the "spend" transaction, if an error occuries in onCallback, the transaction can be rolled back

Example:
```javascript
// example of how to use wallet.spend
var itemToBePurchased = 'test.item.1000';
var cost = 1000; // this is the amount that will be taken out of wallet 'hc'
var hc = gracenode.wallet.create('hc');
hc.spend(userId, cost, itemIdToBePurchase,

        // this callback will be called BEFORE the commit of "spend"
        function (continueCallback) {

                // give the user what the user is spending value for
                user.giveItemByUserId(userId, itemToBePurchased, function (error) {
                        if (error) {
                                // failed to give the user the item
                                return continueCallback(error); // rollback
                        }

                        // succuessfully gave the user the item
                        continueCallback();

                });
        },

        // this callback is to finalize "spend" transaction
        function (error) {
                if (error) {
                        // error on finalizing the transaction
                }

                // we are done!
        }

);

```

> **batch**
<pre>
void batch(Function workCb, Function onFinished)
</pre>
> Batches several add/spend calls in the same transaction
>> "workCb" takes a `function(Batch batch, Function callback)`
>>
>> "onFinished" is a function that takes an error
>>
>> batch has the same add/addFree/addPaid/spend methods as the
>> normal wallet object with onCallback removed, also errors coming
>> from the wallet itself are catched early and sent to the onFinished
>> callback when they happen

Exemple:
```javascript
var hc = gracenode.wallet.create('hc');

hc.batch(function(batch, callback) {
	async.series([
		batch.addPaid.bind(batch, "receipt", "userId", 200, 10),
		batch.spend.bind(batch, "usedId", 15, "something"),
		doSomeOtherMySQLCalls.bind(null, "something")
	], callback);
}, function(error) {
	// do things with error, at this point the transaction is closed
});
```
