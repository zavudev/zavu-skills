---
name: phone-numbers
description: Search, purchase, and manage phone numbers with regulatory compliance and sender assignment.
---

# Phone Numbers

## When to Use

Use this skill when building code to search for, purchase, or manage phone numbers. Covers available number search, purchasing, regulatory requirements, and sender assignment.

## Search Available Numbers

```typescript
const result = await zavu.phoneNumbers.available.list({
  countryCode: "US",
  type: "local",
  contains: "555",
  limit: 10,
});

for (const number of result.items) {
  console.log(number.phoneNumber);       // +15551234567
  console.log(number.friendlyName);      // (555) 123-4567
  console.log(number.locality);          // San Francisco
  console.log(number.capabilities);      // { sms: true, voice: true, mms: true }
  console.log(number.pricing.monthlyPrice); // 1.25
  console.log(number.pricing.isFreeEligible); // true (first US number is free)
}
```

**Python:**
```python
result = zavu.phone_numbers.available.list(
    country_code="US",
    type="local",
    contains="555",
    limit=10,
)
for number in result.items:
    print(number.phone_number, number.pricing.monthly_price)
```

**Go:**
```go
result, err := client.PhoneNumbers.SearchAvailable(context.TODO(), zavudev.PhoneNumberSearchParams{
    CountryCode: zavudev.String("US"),
    Type:        zavudev.String("local"),
    Contains:    zavudev.String("555"),
    Limit:       zavudev.Int(10),
})
for _, number := range result.Items {
    fmt.Println(number.PhoneNumber, number.Pricing.MonthlyPrice)
}
```

**Ruby:**
```ruby
result = client.phone_numbers.search_available(country_code: "US", type: "local", contains: "555", limit: 10)
result.items.each { |number| puts "#{number.phone_number} #{number.pricing.monthly_price}" }
```

**PHP:**
```php
$result = $client->phoneNumbers->searchAvailable([
    'countryCode' => 'US', 'type' => 'local', 'contains' => '555', 'limit' => 10,
]);
foreach ($result->items as $number) {
    echo $number->phoneNumber . ' ' . $number->pricing->monthlyPrice . "\n";
}
```

## Purchase Phone Number

```typescript
const result = await zavu.phoneNumbers.create({
  phoneNumber: "+15551234567",
  name: "Primary Line",
});
console.log(result.phoneNumber.id);     // pn_abc123
console.log(result.phoneNumber.status); // "active"
```

**First US number is free** for each team.

## Phone Number Types

| Type | Description |
|------|-------------|
| `local` | Local phone number |
| `mobile` | Mobile number |
| `tollFree` | Toll-free number |

## Manage Phone Numbers

```typescript
// List owned numbers
const numbers = await zavu.phoneNumbers.list({ status: "active" });
for (const pn of numbers.items) {
  console.log(pn.id, pn.phoneNumber, pn.status);
}

// Get details
const pn = await zavu.phoneNumbers.get({ phoneNumberId: "pn_abc123" });

// Rename
await zavu.phoneNumbers.update({
  phoneNumberId: "pn_abc123",
  name: "Support Line",
});

// Assign to sender
await zavu.phoneNumbers.update({
  phoneNumberId: "pn_abc123",
  senderId: "snd_abc123",
});

// Unassign from sender
await zavu.phoneNumbers.update({
  phoneNumberId: "pn_abc123",
  senderId: null,
});

// Release number (must not be assigned to a sender)
await zavu.phoneNumbers.delete({ phoneNumberId: "pn_abc123" });
```

## Regulatory Requirements

Some countries require additional documentation before phone numbers can be activated:

```typescript
// Check requirements for a country
const requirements = await zavu.phoneNumbers.requirements.list({
  countryCode: "DE",
  type: "local",
});

for (const req of requirements.items) {
  console.log(req.countryCode, req.phoneNumberType, req.action);
  for (const rt of req.requirementTypes) {
    console.log(`  ${rt.name}: ${rt.type} - ${rt.description}`);
  }
}
```

### Requirement Types

| Type | Description |
|------|-------------|
| `textual` | Text field (name, business name) |
| `address` | Physical address |
| `document` | Identity document (passport, ID, etc.) |
| `action` | Action to perform |

### Create Regulatory Address

```typescript
const address = await zavu.addresses.create({
  firstName: "John",
  lastName: "Doe",
  streetAddress: "123 Main St",
  locality: "Berlin",
  postalCode: "10115",
  countryCode: "DE",
});
console.log(address.address.status); // "pending" -> "verified"
```

### Upload Regulatory Document

```typescript
// 1. Get upload URL
const upload = await zavu.documents.uploadUrl();

// 2. Upload file to the presigned URL (use fetch/axios)
await fetch(upload.uploadUrl, {
  method: "PUT",
  body: fileBuffer,
  headers: { "Content-Type": "image/jpeg" },
});

// 3. Create document record
const doc = await zavu.documents.create({
  name: "Passport Scan",
  documentType: "passport",
  storageId: "kg2abc123...",
  mimeType: "image/jpeg",
  fileSize: 102400,
});
console.log(doc.document.status); // "pending" -> "verified"
```

### Document Types

| Type | Description |
|------|-------------|
| `passport` | Passport |
| `national_id` | National ID card |
| `drivers_license` | Driver's license |
| `utility_bill` | Utility bill |
| `tax_id` | Tax ID document |
| `business_registration` | Business registration |
| `proof_of_address` | Proof of address |
| `other` | Other document |

## 10DLC (US A2P Messaging)

For US application-to-person messaging at scale, you may need 10DLC registration:

1. **Brand Registration** - Register your business identity
2. **Campaign Registration** - Register your messaging use case
3. **Number Assignment** - Assign registered numbers to campaigns

10DLC registration is managed through the Zavu dashboard. Contact support for high-volume US messaging requirements.

## Constraints

- First US phone number is free per team
- Phone number name: max 100 characters
- Phone numbers must be unassigned from senders before release
- Regulatory addresses and documents go through verification (pending -> verified/rejected)
- Country code: 2-letter ISO format (e.g., `US`, `DE`, `BR`)
- Search results: max 50 per request
- Some countries require verified address + document before number activation
