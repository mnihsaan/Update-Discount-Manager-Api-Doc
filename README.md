# Update Discount Manager Data Contracts

This document describes the data contracts for the endpoint of update discount manager after payment settlement.


### Endpoint Details
- **URL**: `/payment/settle`
- **Method**: `POST`
- **Content-Type**: `application/json`

### Request Body

The request body should be a JSON object with the following fields:

| Field | Type | Required | Description | Validation Rules |
|-------|------|----------|-------------|------------------|
| `id` | Long | Yes | Unique identifier for the invoice | Must be positive integer |
| `cxPayRef` | Long | Yes | Customer payment reference | Must be positive integer |
| `invoiceLines` | Array | Yes | List of invoice line items | Must contain at least one line item |

#### Invoice Line Item Structure

Each invoice line item should have the following structure:

| Field | Type | Required | Description | Validation Rules |
|-------|------|----------|-------------|------------------|
| `type` | String | Yes | Type of line item | Values like: "discount", "product", "payment" |
| `data` | Object | Yes | Line item data | Must be a valid JSON object |

#### Discount Line Item Data Structure

When `type` is "discount", the `data` object should contain:

| Field | Type | Required | Description | Validation Rules |
|-------|------|----------|-------------|------------------|
| `details` | Object | Yes | Discount details | Must contain either criteria_id or discount_code |
| `details.criteria_id` | Long | Conditional | Discount criteria identifier | Required if discount_code is not provided |
| `details.discount_code` | String | Conditional | Discount code | Required if criteria_id is not provided |

### Response

#### Success Response
**Status Code**: 200 (OK)

| Field | Type | Description |
|-------|------|-------------|
| `invoice_id` | Long | Processed invoice identifier |
| `criteria_id` | Long | Applied discount criteria identifier (if any) |
| `discount_code` | String | Applied discount code (if any) |
| `payid` | Long | Customer payment reference |
| `status` | Integer | Processing status code (200 for success) |
| `message` | String | Processing result message |

#### Error Response
**Status Code**: 400 (Bad Request), 500 (Internal Server Error)

| Field | Type | Description |
|-------|------|-------------|
| `invoice_id` | Long | Invoice identifier (if available) |
| `criteria_id` | Long | Discount criteria identifier (if available) |
| `discount_code` | String | Discount code (if available) |
| `payid` | Long | Customer payment reference (if available) |
| `status` | Integer | HTTP status code |
| `message` | String | Error description |

---

## Data Models

### InvoiceSettlementRequestModel

```json
{
  "id": 12345,
  "cxPayRef": 789012,
  "invoiceLines": [
    {
        "type": "discount",
        "data": {
            "details": {
                "criteria_id": 67,
                "discount_code": "SUMMER2024"
            }
        }
    },
    {
        "type": "product",
        "data": {
            "details": {
                "listing_fee": {
                    "amount": 3500,
                    "variant_id": "297"
                }
            },
            "product_id": "listing_fee_297",
            "sale_channel": "lms"
        }
    }
  ]
}
```

### InvoiceSettlementResponseModel

```json
{
  "invoice_id": 12345,
  "criteria_id": 67,
  "discount_code": "SUMMER2024",
  "payid": 789012,
  "status": 200,
  "message": "Invoice processed successfully. Payid removed from applicable_list. Logged to received_invoice_info."
}
```

---

## Database Entities

### ReceivedInvoiceInfoEntity

This entity stores information about processed invoices for audit and tracking purposes.

| Field | Type | Nullable | Description |
|-------|------|----------|-------------|
| `id` | Long | No | Primary key (auto-generated) |
| `invoice_id` | Long | No | Invoice identifier |
| `criteria_id` | Long | Yes | Applied discount criteria identifier |
| `discount_code` | String | Yes | Applied discount code |
| `payid` | Long | Yes | Customer payment reference |
| `updated_at` | DateTime | Yes | Last update timestamp |

---

## Error Handling

### Error Scenarios

#### 1. Missing Discount Line
- **Condition**: Invoice contains no line items with type "discount"
- **Response**: Status 200 with message "Invoice has no discount. Skipped."

#### 2. Invalid Discount Line Format
- **Condition**: Discount line missing required "details" field
- **Response**: Status 400 with message "Invalid discount line format"

#### 3. Missing Discount Information
- **Condition**: Neither criteria_id nor discount_code provided in discount details
- **Response**: Status 400 with message "Neither criteria_id nor discount_code found"

#### 4. Discount Criteria Not Found
- **Condition**: Specified criteria_id or discount_code not found in database
- **Response**: Status 200 with message "Discount criteria not found in database. Logged to received_invoice_info."

#### 5. Inactive Discount Criteria
- **Condition**: Discount criteria exists but is marked as inactive
- **Response**: Status 200 with message "Discount criteria is currently inactive. Logged to received_invoice_info."

#### 6. Missing Customer Pay Reference
- **Condition**: Discount criteria missing customer_pay_ref field
- **Response**: Status 200 with message "customer_pay_ref missing for criteria. Logged to received_invoice_info."

#### 7. Invalid Applicable List Format
- **Condition**: customer_pay_ref.applicable_list is not an array
- **Response**: Status 200 with message "applicable_list is not an array for criteria. Logged to received_invoice_info."

#### 8. Payid Not in Applicable List
- **Condition**: Specified payid not found in applicable_list
- **Response**: Status 200 with message "Payid not found in applicable_list. Logged to received_invoice_info."

#### 9. Unexpected Errors
- **Condition**: Any unhandled exception during processing
- **Response**: Status 500 with message "Unexpected error occurred"

---

## Processing Flow

1. **Request Validation**: Validate request structure and required fields
2. **Discount Line Detection**: Find line items with type "discount"
3. **Discount Information Extraction**: Extract criteria_id or discount_code from discount line
4. **Audit Logging**: store invoice id, criteria id, payid and discount code in 'received_invoice_info' table. 
5. **Criteria Lookup**: Find discount criteria in database
6. **Customer Pay Reference Update**: Remove payid from applicable_list
7. **Response Generation**: Return processing results with appropriate status and message

