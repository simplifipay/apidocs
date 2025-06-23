# SimpliFi API Documentation

Welcome to the **SimpliFi Platform**! This repository contains the OpenAPI 3.1.0 specification for the SimpliFi
platform, which provides access to various APIs designed to manage card programs, funding sources, users,
transactions, and more.

## Overview

The SimpliFi Platform facilitates seamless financial operations for businesses by offering services such as
creating and managing card programs, issuing cards, handling transactions, and managing user accounts. The APIs in
this repository are well-documented, enabling software teams to integrate the platform into their systems quickly.

This API specification consists of structured paths, schemas, and features to help developers interact efficiently
with the platform. Below is a summary of the key API groups included:

### Core API Groups

1. **Auth API**
   - Handles platform authentication and provides access tokens for secure API requests.

2. **Card Program API**
   - Manages card programs by grouping cards with similar features, authorization controls, and configurations. You
     can create, update, activate, or manage card programs using these APIs.

3. **Funding Source API**
   - Manages funding sources associated with card programs. A funding source serves as the account holding funds
     in a specific currency.

4. **User API**
   - Manages users within the system. Includes functionalities for user creation, updating, retrieval,
     and KYC integration.

5. **Cards API**
   - Manages card life cycle processes including issuance, activation, renewal, loading, and unloading funds.

6. **Transaction API**
   - Provides insights into transactions for cards and accounts, enabling advanced reporting and analytics.

7. **Document API**
   - Manages documents used in KYC and compliance processes. Supports file uploads for proof of funds or
     verification purposes.

8. **Fee API**
   - Provides clients to configure and apply flexible transaction fees at various levels within their card programs.

---

## Key Features

- **Authentication and Authorization**: Secure access to the API through bearer tokens.
- **Manage Card Programs**: Create and control configurations for a set of cards, including transaction limits,
  fee settings, and validation rules.
- **Funding Source Management**: Maintain funding pools for different currencies linked with card programs.
- **Card Operations**: Issue virtual or physical cards, manage their lifecycles, and configure their use cases.
- **Transaction Tracking**: Retrieve historical transaction data on cards or funding sources.
- **User-Based Operations**: Manage cardholders and their KYC information.
- **Document Handling**: Attach and verify documents required for platform usage in compliance-focused workflows.
- **Fee Operations**: Define fee configurations based on transaction types and associate them with specific card program templates.
---

## API Details

This API provides a wide range of support for constructing a card management platform, including schema definitions
and request-response standards. The detailed list of endpoints is categorized by functionality:

- **Authentication**:
  - `/v1/auth/login/{companyName}`
- **Card Program Management**:
  - Create, retrieve, or update card programs under `/v1/card-program/{uuid}`
  - Manage funding sources and balances using `/v2/card-program/{uuid}/funding-source`
  - Transfer funds or raise funding for a program via `/v1/card-program/transfer-fund`
- **User Operations**:
  - `/v1/user` to create, retrieve, update, or deactivate cardholders.
- **Card Operations**:
  - Basic management, e.g., `/v1/card/{uuid}/activate`
  - Transactions and funds, e.g., `/v1/card/{uuid}/load`
  - Renewal or status modifications, e.g., `/v1/card/{uuid}/status`
- **Transactions**:
  - List all transactions via `/v1/transaction`
  - Retrieve statements for funding accounts or cards through paths like `/v1/statement/card/{uuid}`
- **Documents**:
  - Upload files via `/v1/document/upload`
  - Link documents to user onboarding or funding workflows.
- **Fee**:
  - Set up transaction fee structures via `/v1/fees`
  - Add fee events to a queue for specific cards via `/v1/fees/application/card/{cardUuid}`
  - Fees queues for specific cards can be applied via `/v1/fees/application/card/{cardUuid}/apply`

---

## Getting Started

1. **Authentication**:
   - You will need valid `client_id` and `client_secret` to authenticate.
   - Use the `/v1/auth/login/{companyName}` endpoint to obtain a JWT access token.

2. **Integration**:
   - Each request should include the bearer token in the `Authorization` header.
   - Review the schemas for required attributes in the request payloads.

3. **Testing**:
   - Use tools such as **Postman**, **Swagger-UI**, or any API client to explore the endpoints in
     the OpenAPI specification.
   - The provided schemas ensure compatibility for both JSON and form-data inputs.

4. **Testing and Debugging**
    - Validate request payloads and response data while testing on development or staging environments.
---

## External Resources

- **Terms and Conditions**: [Read here](https://simplifipay.com/terms-and-conditions/)
- **Support Email**: [support@simplifipay.com](mailto:support@simplifipay.com)
- **More Information**: [Visit our website](https://simplifipay.com/)

---

## Feedback and Support

Should you have any questions, ideas, or concerns regarding the SimpliFi API, please feel free to contact the
support team via the email provided above. Happy integrating!
