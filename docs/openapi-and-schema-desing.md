# Contract-First Archotecture: Governing Microservices Boundaries with OpenAPI 3.2 and Spectral

## Introduction:

When your microservices drift from their specification you lose a degree of state prediction across microservice 
boundaries. The contract itself is not just any documentation artifact, it is a system that holds your entire architecture together. 

With this notion in mind, there are two major players in the field. The first follows a
Code-first way of doing things, and the other, Contract-first. The priority of the Code-first approach is 
implementation over specification. In other words, write and run the code first before we define the 
contract it actually needs. This approach lays more empahsis on code being the primary source of
truth. On the other hand, the Contract-first design starts by defining an API contract before any code 
is written. 

Both employ or require some sort of business logic, but a core bottleneck for the 
Code-first approach is that it forces downstream consumers (mobile apps, frontends, misc. services) 
to reverse engineer runtime behaviour from implementation code. While the Contract-first provides 
an API specification that operates as a compile-time type system for your entire distributed 
infrastructure. 

Defining a contract first for your API ensures that each rule, schema and definition matches and adheres to your systes businees logic. 

## OpenAPI 3.2 as a Protocol Schema

One benefit of OAS is strict schema. Strict schema modeling helps protect system state by constraining the shape and values of data that can cross an API boundary. Think of an ApI as a controlled doorway into your application's 
state. 

Your contract has state like: 

```json
{
 "accountStatus": "active",
 "balance": 50000, 
 "currency": "ZAR"
} 
```
An API request can potentially change this state. For example: 

```bash
POST /accounts/005/withdraw
```

If your request scheme is poorly defined: 

```yaml
schema:
  type: object
```

Then your API has not really specified what constitutes a valid withdrawal request.

A strict schema might say: 

```yaml
schema:
  type: object
  properties:
    amount:
      type: integer
      minimum: 100
    currency:
      type: string
      enum: [ZAR, USD, EUR]
    required:
      - amount
      - currency
```

Now your API has a contract which says for a withdrawal to be successful and verified, it must 
contain an amount and currency. The amount must be >= 100, and the currency must be one of the 
permitted values. The flow of this contract now gives you verified inputs which provides
a predictable application behaviour with fewer invalid state before any permanent change is made.

### Polymorphism and State Modeling

Another important notion from OAS is polymorphism. Polymorphism becomes important when the shape of a contract, in this case, our payments contract below, differs depending on its state. 

For example:

```yaml
PaymentAccount:
  |--pendingAccount
  |--verifiedAccount
  |--suspendedAccount
```

A polymorphic schema can also include a discriminator object, which can be used as a hint to validate the structure of the model based on an anyOf or oneOf definition. The discriminator also aids in serializing, deserializing and validating data. 

You can define an API contract employing polymorphism with oneOf, allOf and a discriminator.

Click [account_payment](../files/account_payment.yaml) to see a polymorphic contract with oneOf only. 

```yaml
components:
  schemas:

    AccountBase:
      type: object
      required:
        - status
      properties:
        status:
          type: string
          enum:
            - PENDING
            - VERIFIED
            - SUSPENDED

    Account:
      oneOf:
        - $ref: '#/components/schemas/Pending'
        - $ref: '#/components/schemas/Verified'
        - $ref: '#/components/schemas/Suspended'
      discriminator:
        propertyName: status
        mapping:
          PENDING: '#/components/schemas/Pending'
          VERIFIED: '#/components/schemas/Verified'
          SUSPENDED: '#/components/schemas/Suspended'

    Pending:
      allOf:
        - $ref: '#/components/schemas/AccountBase'
        - type: object
          properties:
            name:
              type: string
          required:
            - name

    Verified:
      allOf:
        - $ref: '#/components/schemas/AccountBase'
        - type: object
          properties:
            name:
              type: string
            verifiedAt:
              type: string
              format: date-time
          required:
            - name
            - verifiedAt

    Suspended:
      allOf:
        - $ref: '#/components/schemas/AccountBase'
        - type: object
          properties:
            name:
              type: string
            suspendedAt:
              type: string
              format: date-time
          required:
            - name
            - suspendedAt
```

TODO: 
- add comments in contract
- explain what this contract is doing

Strict schema modeling allows the API contract to describe not only the possible states of an entity, but also the data associated with each state. 


```yaml
Verified:
  allOf:
    - $ref: '#/components/schemas/AccountBase'
    - type: object
      properties:
        verifiedAt:
          type: string
          format: date-time
      required:
        - verifiedAt
```

A response like this: 

```json
{
 "status": "VERIFIED"
}
```

Should return as invalid, since `verifiedAt` is required and defined in the API contract.

Versus: 

```json
{
 "status": "VERIFIED"
 "verifiedAt": "2026-09-22T10:30:00Z"
}
```

The schema can validate the representation of a state. Your application logic, on the other hand remains responsible for validating whether a transition betwwen states is permitted. It is also important to not introduce a schema construct merely because its available. Introduce and use one when the data model benefits from it.

### Full JSON Schema Parity

OpenAPI 3.2 schema object is a superset of JSON schema dradt 2020-12. This specification annotates that unless OAS specifically adds semantics, schema object layouts follow JSON schema. 


? In this instance parity means a progression from OAS 3.0 which had a limited OAS-specific schema model, aligned with OAS 3.1 JSON schema 2020-12 to OAS 3.2. So when you are modeling a dynamic instance like:


```json
{
 "type": "payment",
 "amount": 50000,
 "currency": "ZAR",
 "metadata": {...}
 }
 ```

You can use the JSON schema vocabulary to describe the actual set of valid instances rather than merely documenting the approximate shape. But you should also understand that a schema validation does not guarantee that your backend domain logix is correct. It only prevents payloads that violate the schema from tempering with your data. Your application still has to enforce business and state logic.

### RFC 9457 Problem Details

#### What Problem is RFC 9457 Actually Solving?

HTTP status codes are not always sufficient to tell an API consumer what went wrong. 

Suppose you have three services in different regions


```bash
                   Client
                     │
                 API Gateway
                     │
          ┌──────────┼──────────┐
          ↓          ↓          ↓
   South Africa     Europe      US
       Region      Region      Region
          │          │          │
          └──────────┼──────────┘
                     ↓
              Payment Service
```


What if one or all three goes down. 

South Africa returns: 

```json
{
 "error": "payment_failed",
 "message": "payment provider unavailable"
}
```

Europe returns: 

```json
{
 "code": "SERVICE_PROVIDER_DOWN",
 "reason": "upstream unaccessable" 
}
```

And US:

```json
{
 "status": 503,
 "errorMessage": "temporary failure"
}
```

Notice that all three services are communicating roughly the same message but the client now has to understand three different error contracts. RFC 9457 provides a standardized way to describe the resulting problem detail for the consumer, rather than having every API invent its own error format. 

RFC 9457 gives you a common conceptual structure: 

```json
{
  "type": "https://api.example.com/problems/provider-unavailable",
  "title": "Payment provider unavailable",
  "status": 503,
  "detail": "The payment provider is temporarily unavailable.",
  "instance": "/payments/12345"
}
```

The mportant fields to take note of: 

```bash
type      → What kind of problem is this?
title     → Human-readable summary
status    → HTTP status associated with it
detail    → What happened in this particular occurrence
instance  → Which particular occurrence/resource
```

Another important concept is that RFC 9457 allows for additional information. 


```json
{
  "type": "https://api.example.com/problems/provider-unavailable",
  "title": "Payment provider unavailable",
  "status": 503,
  "detail": "Payment provider is temporarily unavailable.",
  "region": "eu-west-1",
  "retryable": true
}
```

The extra fields are not listed by the RFC, they are your API's extension. With problem details, you get a simpler way of handling or defining error messages to consumers. 

The archicture becomes simpler and coherent: 

```bash

Service A ──┐
Seevice B ──┼──→ Problem Details
Service C ──┘          │
                       ↓
                 Simpler structure
                       │
                       ↓
                Client understands
                problem type
                       │
                       ↓
              Resulting errors handled
                    correctly
```

Clearly defined problem details provide a consistent and simpler error definition across systems and sercices; allowing consumers or clients to identify problem types and apply predictable handling of which service or deployment region produced the error.


## Modeling an Enterprise Contract

Comtinuing from our polymorphic payment contract. We can extend the contract and make it more robust:

Click [enterprise_payment_contract](../files/enterprise_payment_contract.yaml) to see full contract.


```yaml
openapi: 3.2.1

info:
  title: Payment API
  version: 1.0.1
  description: >
    Enterprise API contract for managing payment lifecycle states.

paths:

  /v1/payments/{paymentId}/settle:
    post:
      summary: Settle a payment
      operationId: settlePayment

      parameters:
        - $ref: '#/components/parameters/PaymentId'

      responses:

        '200':
          description: Payment successfully settled.
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/SettledPayment'

        '409':
          description: >
            The payment cannot be settled because its current
            state does not permit settlement.
          content:
            application/problem+json:
              schema:
               # $ref: '#/components/schemas/ProblemDetails'
                $ref: '#/componente/schemas/Errors'
                ...
```

TODO:
- what new changes were added, briefly explain
- add comments where needed in the contract
- what benefit does this have, briefly explain


There is now a clear flow of data or information from the API contract to the payment model to a defined domain logic, making the architecture simpler and coherent with our business logic.  

```bash 

                    API CONTRACT
                         │
                         ▼
              ┌─────────────────────┐
              │ Payment state model │
              └─────────────────────┘
                         │
             ┌───────────┼───────────┐
             ▼           ▼           ▼
         UNSETTLED    SETTLED      FAILED
             │
             │ POST /settle
             ▼
       ┌──────────────┐
       │ Domain logic │
       └──────────────┘
             │
       ┌─────┴──────┐
       │            │
       ▼            ▼
    Success       Failure
       │            │
       ▼            ▼
      200        409 / 422 / 503
                    │
                    ▼
        application/problem+json
                    │
                    ▼
             ProblemDetails
                    │
        ┌───────────┼───────────┐
        ▼           ▼           ▼
      type        status      detail
```

An important distinction to know is that OpenAPI describes the contract; it does not perform transitions between state. 

You define: 

```yaml
status:
  const: SETTLED
```

Means that if this instance is represented as a `SettledPayment`, its status must be `SETTLED`. It does not mean OpenAPI will prevent the backend from changing FAILED to SETTLED. This transition belongs to your application's domain logic.

> Note that all five problem-detail members required can be in your API to enforce a strict contract.
> It is not a claim that RFC 9457 requires everyone of them to be present in every error response.


## Automated Governance With Spectral Ruleset

An OAS contract can be defined using the contract-first aproach with the schema and properties defined and doing a specific task. But having these rules clearly stated does not guaranteee that future contributors or reviewers will continue to follow them.

A new endpoint might introduce:

```yaml
'409':
  content:
    application/json:
      schema:
        $ref: '#/components/schemas/Error'
```

While the rest of your API uses: 

```yaml
'409'
  content:
    application/json:
      schema:
        $ref: '#/components/schemas/ProblemDetails'
```

### Human Review vs. Automated Governance

A reviewer looking at a single pull request can spot any invalid changes at first glance. But across fifty or more services and across hundreds of operations, consistency becomes almost impossible to maintain manually. Spectral turns this requirement into an excutable rule that can be run automatically without much external effort. 

A reviewer or contributor can understand why an API uses RFC 9457. But spectral can ask, does every 4xx/5xx response use ProblemDetails?


### Defining a Spectral Ruleset

Lets define a Spectral ruleset for `enterprise_payent` contract. 

Click [Spectral ruleset](../files/.spectral.yaml) for complete ruleset. 

#### Rule 1: The Problem Details Schema

Rule 1 asks a simpler question: does our canonical ProblemDetails contain the required fields? 

```yaml
problem-details-required-fields:
  description: >
    The canonical ProblemDetails schema must contain
    type, title, status, detail, and instance.

  message: >
    ProblemDetails must define type, title, status,
    detail, and instance.

  given: $.components.schemas.ProblemDetails

  severity: error

  then:
    - field: required
      function: schema
      functionOptions:
        schema:
          type: array
          contains:
            const: type

    - field: required
      function: schema
      functionOptions:
        schema:
          type: array
          contains:
            const: title

    - field: required
      function: schema
      functionOptions:
        schema:
          type: array
          contains:
            const: status

    - field: required
      function: schema
      functionOptions:
        schema:
          type: array
          contains:
            const: detail

    - field: required
      function: schema
      functionOptions:
        schema:
          type: array
          contains:
            const: instance
```

TODO:
- What is rule 1 doing?

Some key things to know about Rule 1 ruleset: 
- given: WHERE do I apply the rule?
- then: WHAT must be true or valid?
- function: HOW do I test it?
- severity: WHAT happens if or when it fails?


#### Rule 2: Handle The Entire API Contract

Rule 2 asks: does every 4xx/5xx response reference that schema. 


```yaml
error-response-problem-details:
  description: >
    Every 4xx and 5xx response must reference
    the ProblemDetails schema.

  message: >
    All 4xx and 5xx responses must use
    #/components/schemas/ProblemDetails.

  given: "$.paths[*][*].responses[?(@property >= '400' && @property < '600')]"

  severity: error

  then:
    field: content.application/problem+json.schema.$ref
    function: pattern

    functionOptions:
      match: "^#/components/schemas/ProblemDetails$"

```

TODO:
- what is rule 2 doing?

`$.paths`: ensures that the rule starts at the APIs path. 
`[*]`: looks at every path, while next 
`[*]`: looks at every operation under those paths. 
`.responses`: enters the response, while 
`[?(@property >= '400' && @property < '600')]`: selects the HTTP error response
`content.application/problem+json.schema.$ref`: Inspects the schema reference.


#### Rule 3: Media Type Definition:

Rule 3 asks: does every 4xx/5xx response use application/problem+json? 

```yaml
problem-details-media-type:
  description: >
    Every 4xx and 5xx response must use
    application/problem+json.

  message: >
    Error responses must use the RFC 9457 media type
    application/problem+json.

  given: "$.paths[*][*].responses[?(@property >= '400' && @property < '600')].content"

  severity: error

  then:
    field: application/problem+json
    function: truthy
```

TODO:
- what is rule 3 doing? 
