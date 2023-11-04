---
title: PayPal Integration With Node.js and React
date: 2021-05-24
# description: Guide to emoji usage in Hugo
tags: [nodejs, reactjs]
---

You might want to integrate PayPal to you sites for easy checkout. This article is to help you understand how easy it is to integrate PayPal in you sites. <!--more-->

PayPal workflow can be defined in 3 steps for simplicity purpose

1. Create Payment: When a user clicks on the PayPal button with the payment info like amount and currency PayPal creates a payment and returns a PaymentID for the user.
2. Authorize User: Authorizing the user to pay his payable and return a PayerID.
3. Execute Payment: Deduct the amount to be paid from the authorized user and redirect to success page.

We are going to use create-react-app and express.js for our example.

<p>
To render the paypal button I am going to use <code>@paypal/react-paypal-js</code> package via npm.
</p>

```javascript
import { PayPalScriptProvider, PayPalButtons } from "@paypal/react-paypal-js";

export default function App() {
	return (
		<PayPalScriptProvider options={{ "client-id": "yourclientid" }}>
			<PayPalButtons style={{ layout: "vertical" }} />
		</PayPalScriptProvider>
	);
}
```

Now that our button is rendered successfully we need to configure our back-end server make calls to paypal api. We need to install paypal node.js sdk but you can make calls to paypal api directly. See the documentation for further references [Payments API](https://developer.paypal.com/docs/api/payments/v2/).

Create a Node.js project in your directory, then run the following command to install the PayPal JavaScript SDK.

```bash
npm install @paypal/checkout-server-sdk
```

<p>Create a config file named <code>paypal.js</code> in your config directory and add the following code.</p>

```javascript
"use strict";

/**
 *
 * PayPal Node JS SDK dependency
 */
const checkoutNodeJssdk = require("@paypal/checkout-server-sdk");

/**
 *
 * Returns PayPal HTTP client instance with environment that has access
 * credentials context. Use this instance to invoke PayPal APIs, provided the
 * credentials have access.
 */
function client() {
	return new checkoutNodeJssdk.core.PayPalHttpClient(environment());
}

/**
 *
 * Set up and return PayPal JavaScript SDK environment with PayPal access credentials.
 * This sample uses SandboxEnvironment. In production, use LiveEnvironment.
 *
 */
function environment() {
	let clientId = process.env.PAYPAL_CLIENT_ID;
	let clientSecret = process.env.PAYPAL_CLIENT_SECRET;

	return new checkoutNodeJssdk.core.SandboxEnvironment(
		clientId,
		clientSecret
	);
}

async function prettyPrint(jsonData, pre = "") {
	let pretty = "";
	function capitalize(string) {
		return string.charAt(0).toUpperCase() + string.slice(1).toLowerCase();
	}
	for (let key in jsonData) {
		if (jsonData.hasOwnProperty(key)) {
			if (isNaN(key)) pretty += pre + capitalize(key) + ": ";
			else pretty += pre + (parseInt(key) + 1) + ": ";
			if (typeof jsonData[key] === "object") {
				pretty += "\n";
				pretty += await prettyPrint(jsonData[key], pre + "    ");
			} else {
				pretty += jsonData[key] + "\n";
			}
		}
	}
	return pretty;
}

module.exports = { client: client, prettyPrint: prettyPrint };
```

Now we need to create an endpoint to create our payment through api call to paypal.

```javascript
const express = require("express");
const bodyParser = require("body-parser");
const cors = require("cors");
const app = express();
const port = 4000;

const checkoutNodeJssdk = require("@paypal/checkout-server-sdk");
const payPalClient = require("./config/paypal");

// parse application/x-www-form-urlencoded
app.use(bodyParser.urlencoded({ extended: false }));

// parse application/json
app.use(bodyParser.json());
app.use(cors());
app.options("*", cors());

app.get("/", (req, res) => {
	res.send("Hello World!");
});

app.post("/create-payment", async (req, res) => {
	try {
		// Construct a request object and set desired parameters
		// Here, OrdersCreateRequest() creates a POST request to /v2/checkout/orders

		// 3. Call PayPal to set up a transaction
		const payReq = new checkoutNodeJssdk.orders.OrdersCreateRequest();
		payReq.prefer("return=representation");
		payReq.requestBody({
			intent: "CAPTURE",
			purchase_units: [
				{
					amount: {
						currency_code: "USD",
						value: "0.01", // your order value
					},
					shipping: {
						name: {
							full_name: "yourname",
						},
						address: {
							address_line_1: "street",
							admin_area_2: "state",
							country_code: "US",
							postal_code: "zipcode",
						},
					},
				},
			],
		});

		let order;
		try {
			order = await payPalClient.client().execute(payReq);
			// console.log(`Order: ${JSON.stringify(order)}`);
		} catch (err) {
			// 4. Handle any errors from the call
			console.error(err);
			return Boom.badRequest(err);
		}

		// 5. Return a successful response to the client with the order ID
		return res.status(201).json({
			orderID: order.result.id,
		});
	} catch (err) {
		return res.status(400).json({
			error: err.toString(),
		});
	}
});

app.listen(port, () => {
	console.log(`Example app listening at http://localhost:${port}`);
});
```

In our client application we can easily make calls to our server to create a payment. Make the following changes to the react app that we created earlier.

```javascript
...

const createOrder = async () => {
    const response = await fetch(`http://localhost:4000/create-payment`, {
        method: "POST",
        headers: {
            'Content-Type': 'application/json'
        },
        body: JSON.stringify({
            amount: {
                currency_code: "USD",
                value: "0.01"
            },
        })
    })

    const {orderID} = await response.json()
    return orderID;
}

return (
    <PayPalScriptProvider
        options={{ "client-id": "yourclientid" }}>
        <PayPalButtons
    		style={{ layout: "vertical" }}
    		createOrder={createOrder}
    	/>
     </PayPalScriptProvider>
);

...
```

We'll receive a orderID in the api call. Which we'll need for the execution of the payment. Don't panic if you get any erros in the console. Now we need to configure our server to authorize the user to make payment and capture the fund.

```javascript
app.post("/capture-payment", async (req, res) => {
	try {
		// 2a. Get the order ID from the request body
		const { orderID } = req.body;

		// 3. Call PayPal to capture the order
		const payReq = new checkoutNodeJssdk.orders.OrdersCaptureRequest(
			orderID
		);
		payReq.requestBody({});

		let capture;
		try {
			capture = await payPalClient.client().execute(payReq);
			// 4. Save the capture ID to your database. Implement logic to save capture to your database for future reference.
			// const captureID = capture.result.purchase_units[0]
			//     .payments.captures[0].id;
			// await database.saveCaptureID(captureID);
		} catch (err) {
			// 5. Handle any errors from the call
			console.error(err);
			return res.status(400).json({
				error: err.toString(),
			});
		}
		// 6. Return a successful response to the client
		return res.status(200).json({
			captureID: capture.result.purchase_units[0].payments.captures[0].id,
		});
	} catch (exp) {
		return res.status(400).json({
			error: err.toString(),
		});
	}
});
```

The new endpoint will capture the fund from users account and deduct it. We will pass the orderID in the request body of this endpoint from our client application.

```javascript
...

	const onApprove = async (data, actions) => {

		const { orderID } = data;

		const response = await fetch(`http://localhost:4000/capture-payment`, {
			method: "POST",
			headers: {
				'Content-Type': 'application/json'
			},
			body: JSON.stringify({ orderID })
		})

		const { captureID } = await response.json()

		// save the captureID to track the order
	}

    return (
		<PayPalScriptProvider options={{ "client-id": "yourclientid" }}>
			<PayPalButtons
				style={{ layout: "vertical" }}
				createOrder={createOrder}
				onApprove={(data, actions) => onApprove(data, actions)}
			/>
		</PayPalScriptProvider>
	);

...
```

That is a quick and easy way to add paypal easy checkout in your application. I will discuss how to set up PayPal Recurring Payments in some other article.
