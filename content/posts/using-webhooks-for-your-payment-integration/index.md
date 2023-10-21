---
title: Benefits of using Webhooks in Payment Integration
date: 2022-01-21
tags: ["backend"]
image: md-alt.png
---

#### What are webhooks?

Webhooks, an approach used to handle unpredictable state of an event. It is one of the techniques used to design `event driven` api.

Let's say we need a status of a perticular resource. What we do in this scenario is we make a request to the api, the api might retrive the information interacting with a database then respond back to the client.

What if the resourse is not in a complete state? The client would have to keep sending request until the state is completed. This is highly inefficient right? If the status change is unpredictable then we might lose a lot of our resources both on the client side and on the server.

This is one of the problems that event driven api can solve.

_There are a lot of detailed articles on webhooks, the purpose of this article to see why and when we need use it in a payment integration so that we can handle some cases which might need taken care of depending on the business model. This is not a detailed tutorial on how to use payment methods whith webhooks from the scratch rather give you an overview of the benefits of using webhooks with an example._

##### Some common cases in payment:

1. User might want to pay later.
2. Maybe you are using some bank transfer integration which need a couple of days for verification.
3. You are using a third party payment service and share a payment link to the user.

In these cases you don't know when the user completes the payment or when the checkout session is finished. You will have to manually update the order status after checking your business account or being confirmed from the bank that the payment went through. You will also have to create your transaction record manually for your application.

> Webhooks are essential for scaling your payments integration by helping you process large volumes of business critical events.

Let's use stripe integration for our example and go step by step:

In our typical payment integration with stripe we create a server endpoint and send a post request with the `amount` and `order_id` (can very based on your implementation and business model) like this:

```javascript
app.post("/create-checkout-session", async (req, res) => {
	const session = await stripe.checkout.sessions.create({
		//we can pass a metadata object containing information about the order which can be retrived from the session
		// metadata: {...},
		line_items: [
			{
				price: "{{PRICE_ID}}",
				quantity: 1,
			},
		],
		mode: "payment",
		success_url: `${YOUR_DOMAIN}/?success=true`,
		cancel_url: `${YOUR_DOMAIN}?canceled=true`,
	});

	res.redirect(303, session.url);
});
```

Then we rely on the client to create/update the order and transaction based on the response.

We can handle this with the webhooks without relying on the client by creating a separate endpoint.

```javascript
app.post(
	"/webhook",
	bodyParser.raw({ type: "application/json" }),
	(request, response) => {
		const payload = request.body;
		const sig = request.headers["stripe-signature"];

		let event;

		try {
			event = stripe.webhooks.constructEvent(
				payload,
				sig,
				endpointSecret
			);
		} catch (err) {
			return response.status(400).send(`Webhook Error: ${err.message}`);
		}

		switch (event.type) {
			case "checkout.session.completed": {
				const session = event.data.object;
				// Save an order in your database, marked as 'awaiting payment'
				createOrder(session);

				// Check if the order is paid (for example, from a card payment)
				//
				// A delayed notification payment will have an `unpaid` status, as
				// you're still waiting for funds to be transferred from the customer's
				// account.
				if (session.payment_status === "paid") {
					fulfillOrder(session);
				}

				break;
			}

			case "checkout.session.async_payment_succeeded": {
				const session = event.data.object;
				// The customer’s payment succeeded.
				// Create / Update the order and transaction status to paid
				// Fulfill the purchase...
				fulfillOrder(session);

				break;
			}

			case "checkout.session.async_payment_failed": {
				const session = event.data.object;

				// Send an email to the customer asking them to retry their order
				emailCustomerAboutFailedPayment(session);

				break;
			}
		}

		response.status(200);
	}
);
```

You can save it when the checkout session is created with a `pending` status then update it when the payment is succeeded or you can create it after the payment is complete. It depends on your business needs.

Now to listen to the webhook events we need to setup the Stripe CLI.

To install the Stripe CLI on Linux without a package manager:

1. Download the latest linux tar.gz file from https://github.com/stripe/stripe-cli/releases/latest
2. Unzip the file: tar -xvf stripe_X.X.X_linux_x86_64.tar.gz
3. Move ./stripe to your execution path.

After you have the Stripe CLI installed, run stripe login in the command line to generate a pairing code to link to your Stripe account. Press Enter to launch your browser and log in to your Stripe account to allow access. The generated API key is valid for 90 days. You can modify or delete the key under API Keys in the Dashboard

```bash
stripe login
Your pairing code is: humour-nifty-finer-magic
Press Enter to open up the browser (^C to quit)
```

If you want to use an existing API key, use the --api-key flag:

```bash
stripe login --api-key {{TEST_API_KEY}}
Your pairing code is: humour-nifty-finer-magic
Press Enter to open up the browser (^C to quit)
```

Use the CLI to forward events to your local webhook endpoint using the listen command.

Assuming your application is running on port 4242, run:

```bash
stripe listen --forward-to http://localhost:4242/webhook
```

In a different terminal tab, use the trigger CLI command to trigger a mock webhook event.

```bash
stripe trigger payment_intent.succeeded
```

You should see the follow event in your listen tab:

```bash
[200 POST] OK payment_intent.succeeded
```

You should also see “PaymentIntent was successful!” printed in the terminal tab your server is running.

If you are using some kind of payment method that doesn't give you the payment status immediately you will also be able to handle those events with the webhooks provided by the payment providers.

Even if you are not dealing with delayed payments it is always a good practice to handle the payment events with webhooks in my opinion.
