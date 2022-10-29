# Logistics Sequence Diagrams

```mermaid
sequenceDiagram
    autonumber
    participant Shim

    participant WLPP 

    participant Sync as Sync Consumer API
    participant Stripe
    participant Logistics
    
    rect rgba(0, 255, 0, .1)
        Note right of Shim: Submit Order
        Shim->>+WLPP: Submit Order
        WLPP->>Sync: Order
        Sync->>WLPP: Order confirmed
        WLPP->>Logistics: Request delivery
        Logistics->>WLPP: Delivery confirmed
        WLPP->>Stripe: Capture payment
        Stripe->>WLPP: Payment confirmed
        WLPP->>-Shim: SNS Order Update
    end

```

# Mapping of various Ordermark data elements to [DoorDashDrive data elements](https://drive.doordash.com/docs/v1.0/index.html#operation/DeliveryListPost).

| Accounts Service      | Shim         | WLPP                 | Logistics             | DoorDashDrive             | DDD Comment |
| ----------------      | ----         | ----                 | ---------             | -------------             | ----------- |
| address  |  | place | origin | pickup_address ||
| address/phone || place/phone | primary_contact_phone | pickup_phone_number
| | customer_info |  | destination | dropoff_address ||
| | customer_info/address |  | destination/address | customer/address ||
| |  |  |             | customer/business_name ||
| | customer_info/name |  | destination/name | customer/first + last ||
| | customer_info/email |  |  | customer/email || optional - maybe we should withhold |
| | customer_info/phone_number |  | destination/phone | customer/phone_number ||
| |  |  |  | customer/should_send_notifications | true |
| | subtotal_info/subtotal | subtotal_info/subtotal | order/order_total | order_value ||
| | prepare_by_datetime | || pickup_time | specify either pickup_time or delivery_time. Window times can override this.
| | || delivery_by | delivery_time | confusion over times. Window times can override this.
| |  |  | order/items | items | we will skip|
| |  |  |  | team_lift_required| false |
| |  |  |  | barcode_scanning_required | false |
| name   || place/name        || pickup_business_name | brand name that customer is ordering from
| address/name || || pickup_instructions | put the name on the outside of the building where Dasher is picking up food (B&M name)
| | service_info/special_instructions | service_info/special_instructions | special_instructions| dropoff_instructions
| || || order_volume | ?? It's a numeric metric which will be mapped to order of size small,medium,large,xlarge
| | subtotal_info/surcharges | subtotal_info/surcharges | tip | tip | only a specific driver tip gets passed to DDD |
| | order_id | app_id + order_id || external_delivery_id | A unique identifier across all store locations under a business, defined by the merchant, which can be used to track this specific delivery.
| | order_id | || driver_reference_tag | Necessary? The internal order identifier at the merchant, which the driver can use to pick up the order. One example of a value for this field is the number the cashier hands the customer at a counter-serve restaurant.
| | | Legal Entity Name (source for data?) || external_business_name | (skip) DDD autosets this on the account| |  |  |  | ||
| | | app_id | app_id | external_store_id || unique identifier, defined by you
| |  |  |  | contains_alcohol | (skip) false - make this a parameter at Logistics |
| |  |  |  | requires_catering_setup | (skip) false |
| |  |  | order/bag_count | num_items | (skip) 1 |
| |  |  |  | signature_required | (skip) ?? |
| |  |  |  | allow_unattended_delivery | (skip) ?? |
| |  |  |  | cash_on_delivery | (include) Optional value in cents. Default is None |
| |  |  |  | delivery_metadata | (skip) JSON document that allows adding metadata about the delivery such as item weight, size, etc. |
| |  |  |  | allowed_vehicles | (skip) Items Enum: "car" "bicycle" "walker" |
| |  |  |  | is_contactless_delivery | ?? Allows for order to be dropped off without physically handing delivery to consumer |
| | | || | | |  |  |  | ||
| | | || | | |  |  |  | ||
| | | || | | |  |  |  | ||
| | | || | | |  |  |  | ||
| |  |  | vendor_id | | n/a?? Vestigial from Habitat
| |  |  | order_route_name | | Used to identify delivery provider
| |  |  | dashboard_key | | n/a?? 
| name                  |              | concept_name         | restaurant_name | restaurant_name | brand name that customer is ordering from

- Someone (Consumer API or WLPP) needs to know the amount of a concept's delivery charge and add that delivery charge to the validated order response.  It's in the logistics config currently.
- When a tip comes in from Google, does that tip go to DDD?

# Logistics API Format 2021-04-08
```JSON
{
    "origin": {
        "city": "Mahopac",
        "state": "NY",
        "zip": "10541",
        "address": "947 South Lake Blvd"
    },
    "delivery_by": "2019-09-10T22:00:00Z",
    "primary_contact_phone": "8185551212",
    "vendor_id": "x",
    "app_id": "ordermark-denver-demo",
    "special_instructions": null,
    "restaurant_name": "Test Restaurant",
    "order_route_name": "DoorDash Drive Service",
    "dashboard_key": "6742696100626432",
    "destination": {
        "city": "mahopac",
        "name": "Preston Rohner",
        "zip": "10541",
        "phone": "+18184686867",
        "state": "NY",
        "address": "947 s lake blvd"
    },
    "tip": 3.57,
    "order": {
        "items": "[{\"item_number\": 0, \"item_id\": \"71601778\", \"item_name\": \"BBQ Chicken Katsu Bowl\", \"quantity\": 1}, {\"item_number\": 1, \"item_id\": \"71601991\", \"item_name\": \"Sesame Seared Tuna\", \"quantity\": 1}]",
        "order_total": 39.3,
        "bag_count": 1
    }
}
```

# Logistics Mock Delivery Plan

- Order submitted
  - Possibly add flag whether delivery cancel is acceptable to order_json in "ordermark" element.
- Sync Consumer API sends update
  - request_order_delivery() which checks app_id's logistics config.
    - Hit Logistics and get some ID.
      - Do we really need, or want, to hit Logistics?  Maybe just return simulated response from Logistics.
  - Save LogisticsInformation() to S3
  - Message scheduling
    - Add new SNS+SQS with a delay
    - Lambda handler:
      - Receives SQS message
      - Grabs next message in series, updates dates/times, etc.
      - Posts to Logistics endpoint
      - Publishes SNS with subsequent request in series of events, if there are more events in the series

# NOT GONNA HAPPEN - Order Validation Flow - IN-1273

```mermaid
sequenceDiagram
    autonumber
    participant DTC
    participant WLPP 
    participant CAPI
    participant SNS
    participant PAPI
    participant SHIM as Olo Shim
    participant OLO as Olo Basket Validate

    rect rgba(0, 255, 0, .1)
      Note right of WLPP: Synchronous Validation Request
      DTC->>WLPP: Synchronous HTTP
      WLPP->>DTC: Request Accepted
      WLPP->>WLPP: Async message posted
    end

    rect rgba(0, 0, 255, .1)
      Note right of WLPP: Asynchronous? Validation Processing
      WLPP->>CAPI: HTTP post
      CAPI->>WLPP: Request Accepted
      CAPI->>PAPI: If CAPI was valid, synchronous HTTP
      CAPI->>SNS: If CAPI was invalid, send response
      PAPI->>SHIM: Synchronous HTTP
      SHIM->>OLO: Synchronous HTTP
      OLO->>SHIM: Synchronous Response
      SHIM->>PAPI: 
      PAPI->>SNS: Publish validation response
      SNS->>WLPP: SNS/SQS subscription
      WLPP->>DTC: SNS Message

    end

```

# Complete Order Flow with Payment Intents - Candidate :) 

```mermaid
sequenceDiagram
    autonumber
    participant DTCFE
    participant DTCBE
    participant WLPP
    participant WLPPDS as Data Store (db/s3)
    participant CAPI
    participant SNS
    participant PAPI
    participant SHIM as Olo Shim
    participant OLO as Olo Basket Validate
    participant Dash

    participant Logistics
    participant Stripe
    participant ddd as Delivery Provider (DDD)

    rect rgba(0, 255, 0, .1)
        Note right of DTCFE: Events are set in motion
        rect rgba(0, 180, 0, .2)
            Note right of DTCFE: Initialize a Payment Intent
            DTCFE->>DTCBE: Request a Payment Intent
            DTCBE->>WLPP: Request a Payment Intent + Order
            WLPP->>WLPPDS: Save order
            WLPP->>Stripe: Request a Payment Intent
            Stripe->>WLPP: Return Payment Intent
            WLPP->>WLPPDS: Save Payment Intent

            rect rgba(0, 0, 255, .1)
                Note right of WLPP: Asynchronous validation processing
                WLPP->>CAPI: HTTP post
                CAPI->>WLPP: Request Accepted
                CAPI->>PAPI: If CAPI was valid, synchronous HTTP
                CAPI->>SNS: If CAPI was invalid, send response
                PAPI->>SHIM: Synchronous HTTP
                SHIM->>OLO: Synchronous HTTP
                OLO->>SHIM: Synchronous Response
                SHIM->>PAPI: 
                PAPI->>SNS: Publish validation response
            end

            WLPP->>WLPP: Async message posted
            WLPP->>DTCBE: Return Payment Intent
            DTCBE->>DTCFE: Return Payment Intent

            rect rgba(0, 180, 0, .2)
                Note right of DTCFE: Delegated to Stripe
                DTCFE->>Stripe: Complete payment
                Stripe->>DTCFE: Redirect to returnURL
                DTCFE->>DTCFE: Show confirmation
            end
        end
    end

    rect rgba(180, 180, 180, .2)
        Note right of WLPP: Positive confirmations required from all
            rect rgba(0, 255, 255, .3)
                Note left of WLPPDS: Stripe confirms payment async
                Stripe->>WLPP: Payment successful webhook update
                WLPP->>DTCBE: 
            end

            rect rgba(0, 255, 255, .3)
                Note left of WLPPDS: POS confirms validation async
                SNS->>CAPI: SNS/SQS subscription
                CAPI->>WLPP: HTTP post
                WLPP->>DTCBE: 
            end
    end

    rect rgba(0, 0, 180, .2)
        Note left of WLPPDS: Arrange Logistics
        WLPP->>Logistics: Synchronous post
        Logistics->>WLPP: Logistics confirmed
        WLPP->>DTCBE: 
    end

    rect rgba(255, 0, 0, .3)
        Note left of WLPPDS: WLPP sends order to dash/POS & gets async response
        WLPP->>CAPI: POST order to Ordermark
        CAPI->>Dash: POST order to dash
        Dash->>CAPI: Callback
        CAPI->>WLPP: Callback
        WLPP->>DTCBE: 
    end

    rect rgba(0, 255, 0, .3)
        Note right of DTCFE: After all events have happened
        WLPP->>DTCBE: Successful order
        DTCBE->>DTCFE: Successful order
    end

    rect rgba(0, 0, 180, .2)
        Note right of WLPP: After delivery request was confirmed
        ddd->>SNS: Delivery update events
        SNS->>WLPP: 
        WLPP->>DTCBE: 
    end
```

# Current Order Processing with Olo Validation

```mermaid
sequenceDiagram
    autonumber
    participant DTCFE
    participant DTCBE
    participant WLPP
    participant WLPPDS as Data Store (db/s3)
    participant CAPI
    participant SNS
    participant PAPI
    participant SHIM as Olo Shim
    participant OLO as Olo Basket Validate
    participant Dash

    participant Logistics
    participant Stripe
    participant ddd as Delivery Provider (DDD)

    rect rgba(0, 255, 0, .1)
        Note right of DTCFE: Events are set in motion
        rect rgba(0, 180, 0, .2)
            Note right of DTCFE: Order submission
            DTCFE->>DTCBE: Submit order
            DTCBE->>WLPP: Submit order
            WLPP->>WLPPDS: Save order
            WLPP->>WLPP: Async message posted

            rect rgba(0, 0, 255, .1)
                Note right of WLPP: Asynchronous validation processing
                WLPP->>CAPI: HTTP post
                CAPI->>WLPP: Request Accepted
                CAPI->>PAPI: If CAPI was valid, synchronous HTTP
                CAPI->>SNS: If CAPI was invalid, send response
                PAPI->>SHIM: Synchronous HTTP
                SHIM->>OLO: Synchronous HTTP
                OLO->>SHIM: Synchronous Response
                SHIM->>PAPI: 
                PAPI->>SNS: Publish validation response
            end
        end
    end
    rect rgba(0, 255, 255, .3)
        Note left of WLPPDS: POS confirms validation async
        SNS->>CAPI: SNS/SQS subscription
        CAPI->>WLPP: HTTP post
        WLPP->>DTCBE: 
    end
    rect rgba(0, 0, 180, .2)
        Note left of WLPPDS: Arrange Logistics
        WLPP->>Logistics: Synchronous post
        Logistics->>WLPP: Logistics confirmed
        WLPP->>DTCBE: 
    end
    rect rgba(0, 255, 255, .3)
        Note left of WLPPDS: Finalize payment
        WLPP->>Stripe: Authorize
        Stripe->>WLPP: Auth confirmed
        WLPP->>Stripe: Payment capture
        Stripe->>WLPP: Payment capture confirmed
        WLPP->>DTCBE: 
    end
    rect rgba(255, 0, 0, .3)
        Note left of WLPPDS: WLPP sends order to dash/POS & gets async response
        WLPP->>CAPI: POST order to Ordermark
        CAPI->>Dash: POST order to dash
        Dash->>CAPI: Callback
        CAPI->>WLPP: Callback
        WLPP->>DTCBE: 
    end

    rect rgba(0, 255, 0, .3)
        Note right of DTCFE: After all events have happened
        WLPP->>DTCBE: Successful order
        DTCBE->>DTCFE: Successful order
    end

    rect rgba(0, 0, 180, .2)
        Note right of WLPP: After delivery request was confirmed
        ddd->>SNS: Delivery update events
        SNS->>WLPP: 
        WLPP->>DTCBE: 
    end

```

## Current Order Processing with Olo Validation

`process_validated_order_updates()` - the bullets (e.g., "1a.") in the diagram map to CloudWatch metrics explained in the bullets below the diagram.

```mermaid
flowchart TD
    dtc([DTC])
    wlpp_in([WLPP])
    capi(Sync Consumer API)
    wlpp(WLPP)
    logistics(Logistics)
    auth(Payment Auth)
    capture(Payment Capture)
    notify(End)
    cancel_log(Cancel Logistics)
    cancel_order(Cancel Order)
    capi2(CAPI)

    dtc-- 1a. and 1b. new order sent -->wlpp_in-- 2a. submit validation -->capi
    wlpp_in-- 2b. failed submission-->notify
    capi-- validation response -->wlpp-- 3b. order is invalid -->notify
    wlpp-- 3a. order is valid -->logistics
    logistics-- Failure -->notify
    logistics-- 4c. and 5a. Success -->auth
    auth-- 5c. Failure -->cancel_log
    auth-- 5b. Success -->capture-- Failure -->cancel_log
    capture-- 6a. Success -->capi2-- Failure -->cancel_order -->cancel_log-->notify
    capi2-- Success -->notify

```

These metrics represent the various steps from the flowchart.
```
1a. "Order Received Async"
1b. "Order Async Processing"
2a. "Order Validation Request Received"
2b. "Order Validation Request Failed"
3a. "Order Validation Response Succeeded"
3b. "Order Validation Response Failed"
4a. "Logistics Not Required"
4b. "Logistics Required"
        "Logistics Phone from Address"
        "Logistics Phone from Brand"
        "Logistics Phone from Brand HACK"
4c. "Logistics Arranged"
5a. "Payment Auth Attempt"
5b. "Payment Auth Success"
5c. "Payment Auth Failed"
6a. "Payment Captured"
```
