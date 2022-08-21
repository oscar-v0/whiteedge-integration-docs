# WhiteEdge Deposit & Withdrawal API Docs

## Setup

The API communication is setup through the WhitEdge Admin Portal (to be implemented), where you'll need to specify the following:

- API URL (required)
  - This URL will be used for all requests sent from WhiteEdge to your server.
- HTTP Headers (optional)
  - Any headers specified here will be included in all requests sent from WhitEdge to your server.
  - We **highly** recommend you specify an `Authorization` header (or similar) with a unique value that then gets verified on your server for all incoming requests.


## Error Messages

In some cases, you may want to display error/warning messages to customers if something goes wrong. E.g. a customer has exceeded their depositing threshold of $100 and they now need to go through some KYC/AML steps to increase their limits. In such cases, your API can optionally return a 4xx response and include the following in the response body:

```ts
{
  // The message that will be displayed on the terminal for the customer to see
  displayMessage: string,
}
```

Note: we currently do not have any multi language support.

## Routes

### `GET /v1/meta (optional)`

The purpose of this endpoint is to provide some basic information and config about your API implementation. Currently, it's only used to provide regular expressions for matching your QR code format.

**200 Response Body**
```ts
{
  // Regular expression strings used to validate QR codes scanned by customers before forwarding them to your API
  // If not provided, all QR codes will be accepted
  depositQrCodeRegex: string, // optional
  withdrawalQrCodeRegex: string // optional
}
```

### `POST /v1/transaction/onCreate`

Once a customer has scanned a deposit/withdrawal QR code, a transaction needs to be initiated. The text extracted from the QR code is sent to your API, along with some other transaction details. Your API should verify the text and return a 2xx response to allow the customer to proceed with the transaction.

Note: it's also important that your API implementation correctly manages sessions, i.e. a customer can only have one active transaction at a given time. If sessions aren't managed, there's a chance of users exploiting any possible vulnerabilities.

**Request Body**

```ts
{
  // The type of transaction that is being performed
  type: "deposit" | "withdrawal",

  // Text extracted from the scanned QR code
  qrCodeText: string,

  // A more user friendly transaction ID that can be displayed to the customer
  // This will also be displayed to the customer on the terminal
  transactionReference: string,

  // ID & Serial ID of the terminal that is being used for the transaction
  terminalId: string,
  terminalSid: string,
}
```

**200 Response**

The response must include a unique `transactionId`. This id will be used for subsequent requests concerning this transaction. Your API should be able to identify the transaction via this id.

The response can optionally include customer information. If provided, customer information will be stored on WhitEdge servers. This gives you the ability to filter transactions by customer on the WhiteEdge Amin Portal. The minimum required to allow filtering is to provide `id`, `email` and/or `mobile`.

```ts
{
  transactionId: string,

  // Optional
  customer: {
    id: string,
    email: string,
    mobile: string,
    firstName: string,
    lastName: string
  }
}
```

**4xx Response**

Any 4xx response allows you to display a message on the terminal using the response body. See [Error Messages](#error-messages)

### `POST /v1/transaction/onCancel/:transactionId`

In case a customer cancels a transaction or a transaction is idle (no input received by the customer), it will be cancelled.

**2xx Response**

_No response body required_

### `POST /v1/transaction/deposit/onBanknoteEscrow`

This endpoint will be queried for every banknote that is inserted for the current deposit transaction. It allows you to control whether banknotes get accepted or rejected. **Banknotes will only get accepted if this endpoint responds with a 2xx status**. Any other response status (e.g. 4xx or 5xx) will result in the banknote being rejected.

**Request Body**

```ts
{
  // Transaction ID
  transactionId: string,

  // Information of the banknote inserted
  banknote: {
    // Currency code, e.g. GBP, USD, EUR, etc.
    currency: string,

    // Denomination, e.g. 5, 10, 50, etc.
    denomination: number,
  },

  // An array of the total amount received so far
  // E.g. [{currency: 'GBP', amount: 20}, {currency: 'USD', amount: 100}]
  receivables: {
    currency: string,
    amount: number,
  }[]
}
```

**2xx Response**

_No response body required_

**4xx Response**

Any 4xx response allows you to display a message on the terminal using the response body. See [Error Messages](#error-messages)

### `POST /v1/transaction/deposit/onBanknoteAccepted (optional)`

This endpoint can be optionally implemented and only serves as confirmation that the banknote was accepted. **Any errors returned by this endpoint will be ignored.**

**Request Body**

```ts
{
  // Transaction ID
  transactionId: string,

  // Information of the banknote inserted
  banknote: {
    // Currency code, e.g. GBP, USD, EUR, etc.
    currency: string,

    // Denomination, e.g. 5, 10, 50, etc.
    denomination: number,
  },

  // An array of the total amounts received so far
  // E.g. [{currency: 'GBP', amount: 20}, {currency: 'USD', amount: 100}]
  // Note: this will include the updated array of receivable amounts after the banknote was accepted from escrow
  receivables: {
    currency: string,
    amount: number,
  }[]
}
```

### `POST /v1/transaction/deposit/onComplete`

Once the customer has finished inserting cash and wants to complete their deposit, this endpoint will be queried. This is where your API should credit the customer's account balance (if applicable) and handle any other matters such as sending notifications. The transaction will be treated as successful if a 2xx response status is returned. Any other response status will assume that the transaction failed and a failure message will be displayed to the customer.

**Request Body**

```ts
{
  // Transaction ID
  transactionId: string,

  // An array of the total amounts received
  // E.g. [{currency: 'GBP', amount: 20}, {currency: 'USD', amount: 100}]
  receivables: {
    currency: string,
    amount: number,
  }[]
}
```
**2xx Response**

In some cases, you may want to apply a fee on the amount inserted by the customer, which results in less credit. Or you may even credit the customer's account with a different currency. In such cases, your API can optionally return a `credited` field to indicate the exact amount received by the customer. This will then be displayed in the WhiteEdge Admin Portal. If this field is omitted, the amount credited is assumed to be the inserted amount.

```ts
{
  // Optional
  // An array of amount & currency credited to the customer
  credited: {
    currency: string,
    amount: number,
  }[]
}
```

**4xx Response**

Any 4xx response allows you to display a message on the terminal using the response body. See [Error Messages](#error-messages)

### `POST /v1/transaction/withdrawal/onStart`

Once the customer has confirmed the amount that they wish to withdraw, this endpoint will be queried. If a 2xx response is returned, the dispensing of banknotes will start. This is where your API should handle any requirements/validations checks, for example:
 - Does the user have sufficient balance to perform the withdrawal?
 - Does the user meet KYC/AML requirements to perform this withdrawal?
 - etc.

**Request Body**

```ts
{
  // Transaction ID
  transactionId: string,

  // Currency of the cash that is to be dispensed, e.g. GBP, USD, EUR, etc.
  currency: string,

  // The amount to be dispensed
  amount: number,
}
```

**2xx Response**

_No response body required_

**4xx Response**

Any 4xx response allows you to display a message on the terminal using the response body. See [Error Messages](#error-messages)


### `POST /v1/transaction/withdrawal/onPreBanknoteDispensed`

This endpoint will be queried for each banknote before it is dispensed. The banknote will only be dispensed if this endpoint returns a 2xx response status. Any other status will result in the transaction being terminated and no further banknotes will be dispensed.

If this endpoint returns a 2xx for the last banknote & the banknote was successfully dispensed, the transaction is assumed to be successful.

**Request Body**

```ts
{
  // Transaction ID
  transactionId: string,

  // Currency of the cash that is to be dispensed, e.g. GBP, USD, EUR, etc.
  currency: string,

  // Denomination of the banknote that will be dispensed
  banknoteDenomination: number,

  // The remaining amount that needs to be dispensed (inclusive of the banknote that is about to be dispensed)
  remainingAmount: number,

  // The original amount requested 
  requestedAmount: number,
}
```
**2xx Response**

_No response body required_

**4xx Response**

Any 4xx response allows you to display a message on the terminal using the response body. See [Error Messages](#error-messages)

### `POST /v1/transaction/withdrawal/onPostBanknoteDispensed (optional)`

Once a banknote is dispensed, this endpoint will be queried to confirm that the previous banknote was dispensed by the terminal. **Any errors returned by this endpoint will be ignored.**

**Request Body**

```ts
{
  // Transaction ID
  transactionId: string,

  // Currency of the cash that is to be dispensed, e.g. GBP, USD, EUR, etc.
  currency: string,

  // Denomination of the banknote that was dispensed
  banknoteDenomination: number,

  // The remaining amount that needs to be dispensed (exclusive of the banknote that was dispensed)
  remainingAmount: number,

  // The original amount requested 
  requestedAmount: number,
}
```

### `POST /v1/transaction/withdrawal/onComplete (optional)`

This endpoint is queried once all banknotes have successfully been dispensed.

**Request Body**

```ts
{
  // Transaction ID
  transactionId: string,

  // Currency of the cash that was dispensed, e.g. GBP, USD, EUR, etc.
  currency: string,

  // The total amount that was dispensed
  amount: number,
}
```
