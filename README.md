# Stock Exchange in Go

This project contains code for a workshop where we highlight features of both Go and Aerospike.

The project will run a simple stock exchange with multiple brokers, who can offer parcels of stock for sale and can bid on parcels. The exchange is fully functional for the context of this workshop. The broker is simply scafolding, providing the building blocks for developers to implement the logic in Go with persistence in Aerospike.

## How it Works

**Exchange**

An exchange will faciliate transactions, manage banks for brokers, and tax brokers for penalties.

**Broker**

Will offer parcels stock for sale and make bids to buy parcesl of stock. Each parcel of stock has a quantity, minimum price and duration of the auction.

**Auction**

When a Seller offers a parcel of stock up for sale, it goes to an auction. The auction will run for the duration of the seller's choosing. The highest bid above the minimum price for the parcel will win the auction. If there is winner, then the auction is considered closed.

If no bid meets the parcel's minimum price within in the time aloted, then the auction will be consider cancelled.

**Notifications**

Each offer, bid, and the closing or cancelling of the auction is transmitted as notifications to the brokers. The brokers will use these notification to determine what to do.

**Penalties**

*Not yet implemented*

- The Exchange will tax Brokers for inactivity.
- The Exchange will tax Brokers for requesting Price Lists.


## Protocol

The exchange and brokers will communicate over Web Sockers on HTTP. The message format will be [JSON-RPC](http://json-rpc.org/wiki/specification).

To summarize:

1. Request message is as follows:

		{"method": string, "params": [any...], "id": any}

2. Response message is as follows:

		{"result": any, "error": any, "id": any}

3. Notification message is as follows:

		{"method": string, "params": [any...]}

	Note Event and Request are the same, except the key difference between the two is the `id` field.


## Commands

### Offer Command

When a broker wants to offer to a parcel for sale, it will send:

	{ "method": "Command.Offer", 
	  "params": [{
	    "BrokerId": BROKER_ID,
	    "TTL": OFFER_TTL,
	    "Ticker": TICKER,
	    "Quantity": QUANTITY,
	    "Price": PRICE
	  }],
	  "id": ID
	}

Where:

- `BROKER_ID` is the broker's identifier.
- `OFFER_TTL` the time to live for the offer.
- `TICKER` is the ticker symbol to make an offer on.
- `QUANTITY` is the number of shares to of the ticket the offer is valid for.
- `PRICE` is the price per share on the offer.
- `ID` the opaque value to be used to match the response to the request.

The offer will live in the exchange until either it expires, is cancelled or is accepted.

The response will be either an error or an ok:

	{"result": OFFER_ID, "error": ERROR, id": ID}

Where:

- `OFFER_ID` is the identifier for the offer, generated by the exchange.
- `ERROR` the error, if an error had occurred.
- `ID` the opaque value to be used to match the response to the request.

### Bid Command

When a broker wants to bid on a parcel up for sale, it will send:

	{ "method": "Command.Bid", 
	  "params": [{
	    "OfferId": OFFER_ID, 
	    "BrokerId": BROKER_ID, 
	    "Price": PRICE
	  }], 
	  "id": ID
	}

Where:

- `OFFER_ID` the identifier for the parcel being auctioned.
- `BROKER_ID` is the broker's identifier.
- `PRICE` the amount the bidder is willing to pay.
- `ID` the opaque value to be used to match the response to the request.

The response will be either an error or the generated identifier for the bid:

	{"result": BID_ID, "id": ID}

Where:

- `BID_ID` is the identifier for the bid, generated by the exchange.
- `ERROR` the error, if an error had occurred.
- `ID` the opaque value to be used to match the response to the request.


## Notifications

### Offer Notice

When a Offer is submitted to the exchange, the exchange will broadcast it to all brokers.

	{ "method": "Offer", 
	  "params": {
	    "Id": OFFER_ID,
	    "BrokerId": BROKER_ID,
	    "TTL": OFFER_TTL,
	    "Ticker": TICKER,
	    "Quantity": QUANTITY,
	    "Price": PRICE
	  }
	}

Where:

- `OFFER_ID` is the identifier for the offer, generated by the exchange.
- `BROKER_ID` is the broker's identifier.
- `OFFER_TTL` the time to live for the offer.
- `TICKER` is the ticker symbol to make an offer on.
- `QUANTITY` is the number of shares to of the ticket the offer is valid for.
- `PRICE` is the price per share on the offer.
- `ID` the opaque value to be used to match the response to the request.

### Bid Notice

When a Bid is submitted to the exchange, the exchange will broadcast it to all brokers.

	{ "method": "Bid", 
	  "params": {
	    "Id": BID_ID,
	    "OfferId": OFFER_ID, 
	    "BrokerId": BROKER_ID, 
	    "Price": PRICE
	  }
	}

Where:

- `BID_ID` is the identifier for the bid, generated by the exchange.
- `OFFER_ID` the identifier for the parcel being auctioned.
- `BROKER_ID` is the broker's identifier.
- `PRICE` the amount the bidder is willing to pay.
- `ID` the opaque value to be used to match the response to the request.


### Close Notice

When a Bid wins the auction, a close notice is sent.

	{ "method": "Close",
	  "params": {
	    "Id": BID_ID,
	    "OfferId": OFFER_ID, 
	    "BrokerId": BROKER_ID, 
	    "Price": PRICE
	  }
	}

Where:

- `BID_ID` is the identifier for the bid, generated by the exchange.
- `OFFER_ID` the identifier for the parcel being auctioned.
- `BROKER_ID` is the broker's identifier.
- `PRICE` the amount the bidder is willing to pay.
- `ID` the opaque value to be used to match the response to the request.

### Cancel Notice

When a no bids win the auction, a cancel notice is sent.

	{ "method": "Cancel",
	  "params": OFFER_ID
	  }
	}

Where:

- `OFFER_ID` the identifier for the parcel being cancelled.

