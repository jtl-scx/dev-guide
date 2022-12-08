
> The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED",  "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119.

# Key Features

With the JTL-Channel API, you can:

-  Describe connected Marketplace Data Structure by providing Category and Attribute Data
-  Manage Product and Offer Listings
-  Update Price and Quantity of an Offer Listing
-  Manage Orders
-  Handle the Post Order Process (Return, Refund)

Target Audience:

* Marketplaces who want to connect with Sellers using JTL-Software with an eazyAuction Subscription
* Software Developer who want to build a connection to a Marketplace

# Terminology

- **Channel**: A Channel is defined as a connection to a Marketplace or any external System which can be connected to JTL-Channel API.
- **Seller**: A Seller is a person – identified by an unique ID (SellerId) – who want to offer and sells his good on the connected Channel.
- **Event**: A Event is an action initiated by a Seller. A Channel needs to react to Events in order to create or update an Offer, or to process certain Post Order actions.
- **Seller API**: This is the counterpart for the Channel API. The ERP System JTL-Wawi is connected with the Seller API.

# Development Cycle and Workflow

Sandbox: https://scx-sbx.api.jtl-software.com
Production: https://scx.api.jtl-software.com 

## Prerequisite

In order to access the SCX - API you will need an API Refresh Token. These Tokens are created during a onboarding process together with JTL. If you are interested in connecting your Marketplace with the JTL ecosystem, just get in touch with us.

https://www.jtl-software.de/kontakt

## Development

A Channel Implementation runs in your own Infrastructure. You as a Channel Integrator have the full responsibility to run, manage and secure your application.

## Setup a local JTL-Wawi instance

It may be helpful to use JTL-Wawi ERP System during Development in order to create new Listings or manage orders. 

To connect the ERP System with the Sandbox Environment

Install a JTL-Wawi Version 1.6+. Connect to the MSSQL Server directly

Add Seller's refresh Token - this Token will be created during onboarding process.

```sql
INSERT INTO SCX.tRefreshToken 
(cRefreshToken, nType)
VALUES 
(N'<SellerAPI-RefreshToken>', 1);
```

Switch SCX Host to Sandbox

```sql
UPDATE dbo.tOptions 
SET cValue ='https://scx-sbx.api.jtl-software.com' 
WHERE ckey = 'SCX.URL'
```

Restart JTL-Wawi

## Using the Seller-API directly

In order to test out Workflows and to send Test Data to the Channel API, you can directly use the Seller API. See the Documentation and our Postman Collection in [Links and other Resources](Channel-API.md#Links%20and%20other%20Resources)

# Seller Management

A Channel must manage Seller Accounts by itself. JTL will never be aware of any credentials which are required by an individual Seller to connect to a Marketplace or an external System (including API Credentials).

Each Channel must maintain a SignUp URL and an Update URL. These URLs must point respectively to a Login or Signup Page hosted by the Channel itself. A Seller will create a SignUp or Update Session inside the JTL-Wawi, which will redirect the Seller together with a short-lived and unique SessionId to the Channel's hosted SignUp and Update URLs.

![](seller_signup.png)

<div style="page-break-after: always; visibility: hidden">
\pagebreak
</div>

## Seller SignUp

The aim of the sign-up process is to avoid managing sensitive data access to a marketplace within the client environment or SCX. The channel itself has sovereignty over the access data to the connected marketplace.

- Channel has a SignUp URL
- A sign-up is initiated via the Seller-API
- The JTL-Wawi of the customer account redirects to the SignUp URL (via web browser).
- The destination of the SignUp URL is a website hosted by the channel. On this website the registration process takes place.
- The channel registers and stores a new seller and assigns a unique SellerId.
- The channel reports the generated SellerId together with the SessionId back to SCX.
- SCX stores the SellerId to the channel and reports all further events to the channel with this SellerId.

### API Examples

Create a new SignUp SessionId

```json
// POST /v1/seller/channel/MYCHANNEL

{
  "signUpUrl": "https://www.mychannel.com/?session=demoSessionId123&expiresAt=1646759360",
  "expiresAt": 1646759360
}
```

The Seller is then redirected to the `signUpUrl`.

On the SignUp Page, the Channel must ask for Seller identification. If a Seller is considered as valid and authenticated, the Channel itself must create a unique SellerId and send them together with the SessionId, from the SignUp URL to the Channel API.

**_Note_**: All Events received from the Channel API will have this SellerId. This SellerId is immutable and can not be changed afterwards.

```json
// POST /v1/channel/seller 

{
  "session": "demoSessionId123",
  "sellerId": "1",
  "companyName": "JTL-Software-GmbH"
}
```

<div style="page-break-after: always; visibility: hidden">
\pagebreak
</div>

## Seller Update

From time to time, it may be necessary for a JTL-Account to update the connection to its Seller. As an example, the access data pertaining to the marketplace may have been renewed. In cases like this, the JTL account must be able to update this data on the channel.

- Channel has an Update URL defined.
- An update process is initiated using the Seller API.
- The client of the JTL Account (JTL-Wawi) redirects the User to the update URL
- The target of the update URL is a website hosted by the channel. On this website the update process takes place.
- The channel asks SCX which SellerId belongs to the current update SessionID.
- The channel enables an update of the seller.
- After the update process, the channel can also update the seller on the SCX system.

### API Examples

The Seller update process in initiated by creating an Update URL for the Channel


```json
// PATCH /v1/seller/channel/MYCHANNEL?sellerId=1
 
{
  "updateUrl": "https://www.mychannel.com/update?session=demoUpdateSessionId123&expiresAt=1646759360",
  "expiresAt": 1646759360
}
```

The Seller is then redirected to the Update URL from the Channel, along with a short-lived SessionId.

For security reasons, the SellerId is not part of the Update URL and can be received through a separate call.

```json
// GET /v1/channel/seller/update-session?sessionId=xyz

{
  "sellerId": "1"
}
```

After the update workflow is handed over, the Channel may now update the Seller at Channel API.

```json
// PARTH /v1/channel/seller


{
  "sessionId": "demoUpdateSessionId",
  "isActive": true,
  "companyName": "JTL-Software-GmbH"
}
```

# Seller-Events

One important component of SCX are Seller and Channel Events. Seller Events are emitted by a Seller integration, such as the JTL-Wawi, while Channel Events are emitted by a Channel Integration. Such events are actions created by an actor (either a Seller or a Channel) and may be handled by connected integrations.

A Channel Integration needs to handle the various Seller Events provided by `GET /v1/channel/events` in order to create new Offer listings, and mark orders as paid or shipped.

We recommend calling the Seller Event Endpoint in a regular interval (such as once a minute) and consume all Events available.

When a event is consumed, it must be acknowledged by calling `DELETE /v1/channel/events`. Otherwise the Event will be transmitted again after a timeout.

Events will be transmitted a maximum of 10 times. Afterwards it will be marked as a dead-letter and will be not transmitted again.

## API Examples

Receive Seller Events

```json
// GET /v1/channel/event

{
    "eventList": [
        {
            "id": "63623b997d2c89a4e3e9f3c7",
            "event": {
                "channel": "WAWIDEV001",
                "sellerId": "EA4590MitName"
            },
            "createdAt": "2022-11-02T09:42:49+00:00",
            "type": "Seller:Meta.SellerAttributesUpdateRequest"
        },
        {
            "id": "636280fd498628e9f0b28984",
            "event": {
                "sellerId": "1",
                "offerId": 822,
                "channelCategoryId": "CAT7",
                "quantity": "0",
                "taxPercent": "19",
                "priceList": [
                    {
                        "id": "B2C",
                        "quantityPriceList": [
                            {
                                "quantity": "1",
                                "amount": "19.99",
                                "currency": "EUR"
                            }
                        ]
                    }
                ],
                "title": "Bike rack",
                "channelAttributeList": [
                    {
                        "attributeId": "WAWI-61427_number_category",
                        "value": "954",
                        "group": "0"
                    }
                ],
                "sku": "843609"
            },
            "createdAt": "2022-11-02T14:38:53+00:00",
            "type": "Seller:Offer.New"
        },
	]
}
```

Acknowledge previously received events.

```json
// DELETE /v1/channel/event

{
    "eventIdList": [
	    "63623b997d2c89a4e3e9f3c7",
        "636280fd65a66c4430ec0d67"
    ]
}
```
<div style="page-break-after: always; visibility: hidden">
\pagebreak
</div>

# Listing Process

Within the SCX context there is no concept of a product catalog. Only offer data is transmitted via the SCX interface, this can however contain detailed product data as well- if required by the Channel.

A Channel must provide descriptive Data to describe what an Offer Listing may look like on a connected Marketplace.

## Prices Types

There must be at lead one Price Type available to create a listing on a connected Marketplace. Examples for Price Types are B2C or B2B prices.

### API Example

```json
// POST https://scx-sbx.api.jtl-software.com/v1/channel/price

{
  "priceTypeId": "MarketplaceTypeId",
  "displayName": "Marketplace Price",
  "description": "Selling price on Marketplace"
}
```

The `priceTypeId` will be transmitted with the `Seller:Offer.New` or `Seller:Offer.Update` Seller Events. 

```json
{
  "sellerId": "1",
  "offerId": 4711,
  "channelCategoryId": "Stuff",
  "quantity": "508.00",
  "taxPercent": "19",
  "priceList": [
    {
      "id": "MarketplaceTypeId",
      "quantityPriceList": [
        {
          "amount": "6.95",
          "currency": "EUR",
          "quantity": "1"
        }
      ]
    }
  ],
  "...": "..."
}
```

<div style="page-break-after: always; visibility: hidden">
\pagebreak
</div>

## Category Tree

A connected Channel may provide a Category Tree to set specific Attributes related to a Category.

### API Examples

The API Endpoint is in the process of replacing the entire Category Tree.

```json
// PUT /v1/channel/categories

{
    "categoryList": [
        {
            "categoryId": "1",
            "displayName": "First Category",
            "listingAllowed": false,
            "parentCategoryId": "0"
        },
        {
            "categoryId": "1.1",
            "displayName": "First Leaf Category",
            "listingAllowed": true,
            "parentCategoryId": "1"
        },
        {
            "categoryId": "1.2",
            "displayName": "Second Leaf Category",
            "listingAllowed": true,
            "parentCategoryId": "1"
        }
	]
}
```

<div style="page-break-after: always; visibility: hidden">
\pagebreak
</div>


## Attributes

Attributes provide a very simple but powerful way of describing an offer for a channel. This allows the channel to define all marketplace requirements for an offer by means of attributes.

SCX differentiates between three types of Attributes. Each attribute type share the same general structure, but targets a different use case.

### Global Attributes

Global Attributes should be used when data is required for each Offer.

### Category Attributes

Category Attributes are related to a `categoryId` inside the Category Tree. JTL-Wawi will display these attributes only when the Offer is part of a given Category.

### Seller Attributes

Seller Attributes are Global Attributes and can be used when a Seller requires individual settings for their Offers. 

### Item-Specific Attributes

Item-specific attributes are not specified by the channel, but are instead created and transferred by the merchant. As such, these attributes represent a special form- each attribute must provide a simple, non-schematic, key-value data structure. These Attributes will be transmitted along with the `channelAttributeList` in each OfferNew or OfferUpdate Event.

These attributes are highly individual, and depend on the input of the seller. The channel can use these attributes to provide product and offer data.


### API Examples

Attributes with different Types.

![](scx_attributes_001%201.png)

```json
// POST /v1/channel/attribute/category/DEMO-TYPES

{
    "attributeList": [
        {
            "attributeId": "DEMO-TYPES_smalltext",
            "displayName": "Smalltext",
            "type": "smalltext"
        },
        {
            "attributeId": "DEMO-TYPES_text",
            "displayName": "Text",
            "type": "text"
        },
        {
            "attributeId": "DEMO-TYPES_integer",
            "displayName": "Integer",
            "type": "integer"
        },
        {
            "attributeId": "DEMO-TYPES_decimal",
            "displayName": "Decimal",
            "type": "decimal"
        },
        {
            "attributeId": "DEMO-TYPES_boolean",
            "displayName": "Boolean",
            "type": "boolean"
        },
        {
            "attributeId": "DEMO-TYPES_enum",
            "displayName": "Enum",
            "type": "enum",
            "values": [
                {"display": "Value 1", "value": "Id1", "sort": 10},
                {"display": "Value 2", "value": "Id2", "sort": 20},
                {"value": "This has no Display"}
            ]
        },
        {
            "attributeId": "DEMO-TYPES_htmltext",
            "displayName": "Html-text",
            "type": "htmltext"
        },
        {
            "attributeId": "DEMO-TYPES_date",
            "displayName": "Date",
            "type": "date"
        },
        {
            "attributeId": "DEMO-TYPES_image",
            "displayName": "Image",
            "description": "Not yet supported in JTL-Wawi",
            "type": "image"
        },
        {
            "attributeId": "DEMO-TYPES_document",
            "displayName": "Document",
            "description": "Not yet supported in JTL-Wawi",
            "type": "document"
        }
    ]
}
```

Using Sections / Sub-Sections to organize attributes into logical groups.

![](scx_attributes_003.png)

```json
// POST /v1/channel/attribute/category/DEMO-SECTIONS

{
    "attributeList": [
        {
            "attributeId": "DEMO-SECTIONS_WAREHOUSE",
            "displayName": "Warehouse",
            "type": "smalltext",
            "section": "Shipping",
            "sectionPosition": 100
        },
        {
            "attributeId": "DEMO-SECTIONS_SHIPPING",
            "displayName": "Shipping Group",
            "type": "smalltext",
            "section": "Shipping",
            "sectionPosition": 80
        },
        {
            "attributeId": "DEMO-SECTIONS_LEADTIME",
            "displayName": "Lead time until shipment",
            "type": "integer",
            "section": "Shipping",
            "sectionPosition": 50
        },
        {
            "attributeId": "DEMO-SECTIONS_OFFER_START",
            "displayName": "Start",
            "type": "date",
            "section": "Discount",
            "sectionPosition": 50
        },
        {
            "attributeId": "DEMO-SECTIONS_OFFER_DISCOUNT",
            "displayName": "Discount Value",
            "type": "decimal",
            "section": "Discount",
            "sectionPosition": 100,
            "subSection": "Deduction",
            "subSectionPosition": 100
        },
        {
            "attributeId": "DEMO-SECTIONS_OFFER_DISCOUNT_UNIT",
            "displayName": "Discount Unit",
            "type": "enum",
            "values": [
                {"value": "%"},
                {"value": "EUR"}
            ],
            "section": "Discount",
            "sectionPosition": 100,
            "subSection": "Deduction",
            "subSectionPosition": 90
        }
    ]
}
```


It is also possible to create repeatable attributes, as long as the attribute supports multiple values.

![](scx_attrbutes_002.png)


```json
// POST /v1/channel/attribute/category/DEMO-MULTIPLE_ALLOWED
{
    "attributeList": [
        {
            "attributeId": "DEMO-TYPES_isMultipleAllowed",
            "displayName": "Attribute is Multiple Allowed",
            "isMultipleAllowed": true,
            "type": "smalltext"
        }
    ]
}
```

Repeatable Sub-Sections can be used to multiply an entire Section with multiple Attributes inside.

![](scx_attributes_005.png)

```json
// POST /v1/channel/attribute/category/DEMO-REPEATABLE_SUBSECTIONS

{
    "attributeList": [
        {
            "attributeId": "shipping_carrier",
            "displayName": "Shipping carrier",
            "type": "enum",
            "values": [
                {
                    "value": "carrierID_2",
                    "display": "Letter",
                    "sort": 1
                },
                {
                    "value": "carrierID_3",
                    "display": "DHL Package (Small)",
                    "sort": 2
                },
                {
                    "value": "carrierID_4",
                    "display": "DHL Package",
                    "sort": 3
                },
                {
                    "value": "carrierID_22",
                    "display": "Free Download",
                    "sort": 20
                }
            ],
            "section": "Shipping cost",
            "sectionPosition": 2,
            "subSection": "Shipping method",
            "subSectionPosition": 10,
            "isRepeatableSubSection": true
        },
        {
            "attributeId": "shipping_cost_nat",
            "displayName": "National",
            "type": "decimal",
            "section": "Shipping cost",
            "sectionPosition": 2,
            "subSection": "Shipping method",
            "subSectionPosition": 9,
            "isRepeatableSubSection": true
        },
        {
            "attributeId": "shipping_cost_eu",
            "displayName": "EU",
            "isMultipleAllowed": false,
            "type": "decimal",
            "section": "Shipping cost",
            "sectionPosition": 2,
            "subSection": "Shipping method",
            "subSectionPosition": 8,
            "isRepeatableSubSection": true
        },
        {
            "attributeId": "shipping_cost_int",
            "displayName": "International",
            "isMultipleAllowed": false,
            "type": "decimal",
            "section": "Shipping cost",
            "sectionPosition": 2,
            "subSection": "Shipping method",
            "subSectionPosition": 7,
            "isRepeatableSubSection": true
        }
    ]
}
```

When using repeatable sections, the JTL-Wawi generates a group ID, which is then transferred back to the Channel with the offer. This makes it possible to recognize related attributes in the offer data.

```json
// GET /v1/channel/event

// Note: Simplyfied event 
{
  "type": "Seller:Offer.New",
  "sellerId": "1",  
  "offerId": 1,  
  "channelCategoryId": "DEMO-REPEATABLE_SUBSECTIONS", 
  "channelAttributeList": [  
    {  
      "attributeId": "shipping_carrier",  
      "value": "DHL Package",  
      "group": "1"  
    },  
    {  
      "attributeId": "shipping_cost_nat",  
      "value": "2.99",  
      "group": "1"  
    },  
    {  
      "attributeId": "shipping_cost_eu",  
      "value": "2.99",  
      "group": "1"  
    },  
    {  
      "attributeId": "shipping_cost_int",  
      "value": "9.99",  
      "group": "1"  
    },
    {  
      "attributeId": "shipping_carrier",  
      "value": "Brief",  
      "group": "2"  
    },  
    {  
      "attributeId": "shipping_cost_nat",  
      "value": "1.20",  
      "group": "2"  
    },  
    {  
      "attributeId": "shipping_cost_eu",  
      "value": "2.40",  
      "group": "2"  
    },  
    {  
      "attributeId": "shipping_cost_int",  
      "value": "2.40",  
      "group": "2"  
    },     
  ]  
}
```

# Transmitting Offers

Offers are transmitted by `Seller:Offer.New` and `Seller:Offer.Update` Event.
Each time the Offer Data changes the JTL-Wawi will transmit the full Offer again.

### Essentials Properties 

It is generally not necessary to provide attributes for essential data, as by default each offer already implements a set of essential properties, including a title, description, GTIN, quantity and price.

### Pictures / Images

Product or Offer Pictures are transmitted as a Link along with the Offer Event.
These Links have a time to live of 7 days after the Offer Event was emitted. The filename is generated based on the file content- as such, each seperate picture will have a filename similar to its file content.

## Offer State

![](offer_listing.png)

We reccommend informing the Seller about the processing status of their offers. Most Marketplaces use an asynchronous Listing Process, where the Offer Data is curated in a semi-automated process— a process which may take some time.

### Status: In-Progress

Send this status to inform the Seller that the Offer Listing process for their Offer is now in progress.

```json
// POST /v1/channel/offer/in-progress

{
	"offerList": [
		{
            "offerId": 1,
			"sellerId": "1",
            "startedAt": "2022-11-28T01:00:13+00"
		}
	]
}
```

### Status: Successful

Once an Offer is successfully listed on the connected marketplace, we reccommend informing the Seller. The optional `listingUrl` is useful in this case and will let the Seller to visit and thereby directly check the listing on the connected marketplace.

```json
// POST /v1/channel/offer/listed

{
	"offerList": [
		{
			"sellerId": "1",
			"offerId": 1,
			"listedAt": "2022-11-28T01:00:13+00",
			"listingUrl": "https://marketplace.de/offer1",
			"channelOfferId": "AFGHDHDFH"
		}
	]
}
```

### Status: Failed

If the Offer failed the the listing process, inform the Seller about it. 

```json
// POST v1/channel/offer/listing-failed

{
	"offerList": [
		{
			"sellerId": "1",
			"offerId": 1,
			"errorList": [
				{
					"code": "123",
					"message": "A listing error occurred",
					"longMessage": "A serious listing error occurred with details."
				}
			],
			"failedAt": "2019-02-09T05:33:12+00:00"
		}
	]
}
```

<div style="page-break-after: always; visibility: hidden">
\pagebreak
</div>

# Order Process

* A new Order must be created by calling `POST /v1/channel/order` first
* The OrderId is unique, and can only exist once for a SellerId
* Once an Order is created it can be updated using 
	* `PUT /v1/channel/order/status` to update the status of an Order or Order Item
	* `PUT /v1/channel/order/address-update` to update the Address information

## Order Status

* `CREATED`: Orders with the Status created are not yet ready for shipping. Orders in this status can be used for stock reservation. Once the Order is ready for shipping, the status will change to `ACCEPTED`.
* `UNCACKED`: Orders in this Status are not yet ready for shipping. The Seller must first accept the Order. **_Note_**: The JTL-Wawi does not yet support Order Acknowledgment.
* `ACCEPTED`: Orders with the Status accepted are ready for shipping. 

## OrderItem Status

Each OrderItem has a status, and this status should be used to determine the Order Status itself.

* *`UNSHIPPED`: The OrderItem is ready for shipping, and can be shipped once the Order is `ACCEPTED`.
* `SHIPPED`: The OrderItem is marked as being shipped
* `CANCELED_BY_SELLER`: The OrderItem has been cancelled by the Seller
* `CANCELED_BY_BUYER`: The OrderItem has been cancelled by the Buyer or the Marketplace
* `RETURNED`: The OrderItem has been returned to Seller.
* `REFUNDED`: The OrderItem has been refunded.

## Status transition limitations and rules

* Orders with a Status of `UNACKED` or `CREATED` may not have valid Address Information
* Address Update can only apply to Orders with a Status of `UNACKED` or `CREATED`
* Address Information must be available before a status transition to `ACCEPTED`
* Once an Order is set with the Status `ACCEPTED` the Order Status can not be changed

Order Item Status can convey the following status information

  Stauts | UNSHIPPED | SHIPPED | CANCELED_BY_SELLER | CANCELED_BY_BUYER | RETURNED | REFUNDED
  --- | --- | --- | --- | --- | --- | --- 
  UNSHIPPED           | ✓ | ✓ | ✓ | ✓ | ✓ | ✓      
  SHIPPED             | - | ✓ | ✓ | ✓ | ✓ | ✓        
  CANCELED_BY_SELLER  | - | - | ✓ | - | - | -  
  CANCELED_BY_BUYER   | - | - | - | ✓ | - | -   
  RETURNED            | - | - | - | - | ✓ | ✓      
  REFUNDED            | - | - | - | - | - | ✓     


### API Examples

Create a `CREATED` Order using 

```json
// `POST 'https://scx-sbx.api.jtl-software.com/v1/channel/order`

{
  "orderList": [
    {
      "sellerId": "1",
      "orderStatus": "CREATED",
      "orderId": "OrderId_000001",
      "purchasedAt": "2020-02-25T15:05:20+00:00",
      "lastChangedAt": "2020-02-25T15:05:20+00:00",
      "currency": "EUR",
      "orderItem": [
        {
          "orderItemId": "SHIPPING-0001",
          "type": "SHIPPING",
          "grossPrice": "2.00",
          "taxPercent": "19",
          "quantity": "1.0",
          "shippingGroup": "test"
        },
        {
          "orderItemId": "ABC-0001",
          "type": "ITEM",
          "grossPrice": "19.99",
          "total": "19.99",
          "taxPercent": "19",
          "sku": "1234",
          "channelOfferId": "1",
          "quantity": "1",
          "title": "Jeans (4006680069951)",
          "note": "As a selection"
        },
        {
          "orderItemId": "ABC-0002",
          "type": "ITEM",
          "grossPrice": "19.99",
          "total":"19.99",
          "taxPercent":"19",
          "sku": "ART-WAWI-55070",
          "channelOfferId": "2",        
          "quantity": 1,
          "title": "T-Shirt (ART-WAWI-55070)",
          "note": "As a selection"          
        }
      ]
    }
  ]
}
```

Send Address Information for an order with the status `CREATED`

```json
// PUT 'https://scx-sbx.api.jtl-software.com/v1/channel/order/address-update

{
    "orderList": [
        {
            "orderId": "OrderId_000001",
            "sellerId": "1",
            "billingAddress": {
                "firstName": "Arno",
                "lastName": "Nym",
                "gender": "male",
                "street": "Am Feld",
                "houseNumber": "16",
                "postcode": "123456",
                "city": "Dingenskirschen",
                "country": "DE"
            },
            "shippingAddress": {
                "firstName": "Arno",
                "lastName": "Nym",
                "gender": "male",
                "street": "Am Feld",
                "houseNumber": "16",
                "postcode": "123456",
                "city": "Dingenskirschen",
                "country": "DE"
            }
        }
    ]
}
```

Update Order and set Status to `ACCEPTED`

```json
// PUT 'https://scx-sbx.api.jtl-software.com/v1/channel/order/status

{
	"orderList": [
		{
			"orderId": "OrderId_000001",
			"sellerId": "1",
			"orderStatus": "CREATED"
		}
	]
}
```

Send a Status change for an OrderItem

```json
// PUT 'https://scx-sbx.api.jtl-software.com/v1/channel/order/status

{
	"orderList": [
		{
			"orderId": "OrderId_000001",
			"sellerId": "1",
			"orderStatus": "CREATED",
			"orderItems": [
                {
                    "orderItemId": "ABC-0001",
                    "itemStatus": "SHIPPED",
                    "paymentStatus": "PAID"
                },
                {
                    "orderItemId": "ABC-0002",
                    "itemStatus": "SHIPPED",
                    "paymentStatus": "PAID"
                }
            ]
		}
	]
}
```

<div style="page-break-after: always; visibility: hidden">
\pagebreak
</div>


# Post-Order Processes

## Cancellation (by Seller)

![](order_cancellation.png)

- Sellers send a cancellation request together with a `CancellationRequestId` (UUID) which the client, i.e., the JTL-Wawi, should remember in order to be able to assign the response later.
- The UUID is used for the unique assignment of the data (OrderId / OrderItemIDs) at the client.
- A cancellation should always include all items to be cancelled. If the entire order is cancelled, all order items contained must also be specified. Accordingly, however, only a partial cancellation of individual items can take place.

## Cancellation (by Buyer/Marketplace)

![](order_cancellation_buyer.png)

- Buyers or Marketplaces send a cancellation request together with a `CancellationRequestId` (UUID)
- The UUID is used for the unique assignment of the data (OrderId / OrderItemIDs) at the channel.
- A cancellation should always include all items to be cancelled. If the entire order is cancelled, all order items contained must also be specified. Accordingly, however, only a partial cancellation of individual items can take place.
- Using this Workflow on the Channel side is considered optional- it is also possible to set the Order Status to cancel.

## Return

![](return.png)

### Channel informs about an upcoming return event

**_Note_**: This is a optional step in the Workflow. Not all Marketplaces support a Return Announcement.

- Channel reports via `Channel:Order.Return`  Event that a return process has been initiated on a connected marketplace
	- The following data is provided
		- OrderId
		- OrderItem List (ItemId, Quantity, Reason, Comment)
	- The following data can optionally be provided by the channel
		- ChannelReturnId - internal ID to identify the return case
		- Return Tracking Information (Carrier, TrackingNo.)
- JTL-Wawi stores the upcoming return case
    - if no item return is required, the return can be answered directly via `POST /v1/seller/order/return`
    - if a return is required the Merchant now waits for the incoming return.

### (Optional) Merchant decides that no return is required

**_Note_**: This is a optional step in the Workflow.

The merchant has the option to decide whether the customer does not need to make a return. If no return is required, the `POST /v1/seller/order/return` can be sent by the merchant after receiving the Channel:Order.Return event.
The property `requireReturnShipping`  should be set to `false` here.

### Return shipment arrives at the merchant's warehouse

- Once the return has arrived, the merchant will check the items and confirm the return using `POST /v1/seller/order/return`.
- If a return already exists in JTL Wawi, the information stored there should be used to transfer the return to SCX.
	- This information includes the `channelReturnId`
- Furthermore, information about the individual order items must be transferred
	- The order number (orderId) 
	- The quantity that was returned
	- If the return is accepted (acceptReturn)
	- A condition must be specified
	- A reason can be specified
	- A additional note that can be stored.

## Refund

![](refund.png)

Refunds are initiated directly by the merchant, e.g. after a return has been processed or following an agreement with the buyer (via ticket/email/phone). The channel processes the refund and sends a `Channel:Order.RefundProcessingResult` after a success or failure 
back to the merchant to ensure that a refund has been properly processed. 

# Links and other Resources

Channel-API Documentation
https://scx-sandbox.ui.jtl-software.com/docs/api_channel.html

Seller-API Documentation
https://scx-sandbox.ui.jtl-software.com/docs/api_seller.html

Postman Collection
https://www.postman.com/jtl-eazyauction/workspace/jtl-scx-public

JTL-Guide 
https://guide.jtl-software.de/






