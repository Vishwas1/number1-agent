# Verify & Access Technical Implementation Guide v0\.4

# Introduction 

This guide explains important concepts of Concordium’s Verify & Access service, and provide detailed instructions on how to integrate the solution in a service.

Verify & Access provides an access gateway using verifiable user attributes based on Concordium ID. The system can be used by any service provider (online merchants, content platforms, etc.) to ensure controlled access to ID-gated services, without having to process sensitive user data. In order to enhance regulatory compliance, the solution generates privacy-preserving audit records, which can be verified by independent third parties.

## Solution 

When a user requests access to a restricted service, they provide a **Verifiable Presentation (VP)** from their Concordium ID that proves they meet the relevant requirement (e.g. “Age is over 18”). The merchant’s backend then verifies this proof and, if valid, grants access. Verify & Access is based on Zero-Knowledge Proofs and aligns with the W3C standard, [Verifiable Credentials](https://www.w3.org/TR/vc-overview/).

To make the process auditable, the solution generates a **Verification Request Anchor (VRA)**  transaction on the Concordium blockchain. The VRA contains a hash of the *verification context* and the *credential statements* (specifies the conditions that the user’s ID attributes meet e.g. Age \>18), providing a tamper-evident, timestamped record of the verification request.

In addition to the VRA, a **Verification Audit Anchor (VAA)** is created.  
The VAA provides a tamper-evident, timestamped record of the merchant’s audit data, enabling independent third parties, such as auditors, to confirm that the verification took place and that the required criteria were met, all without exposing any sensitive personal information. 

This approach ensures privacy, enforces eligibility for restricted products, and provides a smooth, secure, and fully auditable verification process.

The end to end flow of “How it works“ is [here](https://sequencediagram.org/index.html#initialData=C4S2BsFMAIDVIE4gGYE8C0BBAxtyBnfaAMXAHsB3aAVXxADsBzaAYTPuzKQBMBXAW2gBJACIAoCQENswLtGi98iMZHrAEqfAAdpDZgEYAdABYxOhKGwgdaxgjK8t0AMTgQjABbA7kVNADcAAzQAFQhALKI2B6SamaSFiBWNsDQkQjRscAA5ETE9mqq3GLO5pbWWcIi6ADKIgDS8YnJlemZarnQAELSANZFTeUprOycCNwgAnCIKCCI0DWIAG5JkCr0xWVJFWojHFwTU6KYWlqD28Ns++OTgl3k2L2ZDBJiivPoAHxpUTEdeQVgEUAFzQAAqSAI0Fk0BweEI0I8MAokAARnQga87A4nAB1STgKCpK70SAyaAABUkICQTGgElRZAAHtAyEt5m0-jkAewgRtQQBxVSISRAogUAlE6Cceik8nUABKQmgsW40HwHkoRAAigrpWRuGs3koENB0F89mNDoJjqdQQB5LSqIjAJFVE5OVXq7CxF1u3X6w0STlZTr5XlFM3fK5W27uu3QfGEyDE9hy1KKoQSGMHOO2pzm74h-4kQEgynU2nMD32dlqiQDenYxzTJDIECSVFQSkIAiqYCikDsaAKyAAR14BGAr2L3NLEY2UZ+GS5nR6j3LYQp9pqYOgAHp9+y2x2u5B0Fpe0o1IP2Ph99he6Lz8fZj7QOx0L2J1OADr0ABvfVZTJD96CENUAF8wnpMRZzXPpIwtHMbimeAT3mRYEBWPBQS3Hc90PV9207KALyvftb3oe9H0gZ90GIpIqK-cdJ3wYB-yAmV0yHcDuAAGmgK8HAySAIMEmUgSZYAanUPRBK0XguySIR6GQMhBPwJSACtQJYcBqX4IhINCEJYLERkWTZeYUOtVtZkw5ZVlBYguCM6AAB5OHIU1nB8VBPnQ2ZSMgCkKJvMDRx-diPIfMgfO+AAKWAKQVABKf9oAAbQoMBNV4VIvPiuRnC7SdPm-NjgDBAANWLvK4T4AF0FDoOlbLjOp6n-dASDcogWF7Q01A7cBZOffh+2gRLFD0dVgH4VJHzAGZJGgEBDBgSRGBgT59AADgy+hetchB3KuKTgFU9SzqohlmVZY9LVzNCZnbRzsOc6AJh29joAAXgWAAJTAACYAFYADYYg1RLKqnLKACJJMgaTEaa-j4fYpHaOG0ACXGoFJrUfB0bS6BMqKnyXDKyBPjCvsIt4kdWKnWEOE1U0RFFSR6uKhBPkBoDEeAVAnURvCQhYFgRFgBVMDCfjEePOh2Al6B9EExGYY8CWfqnSDXg616MNNLCcJgZDRheu4HieGIGFBCkyD+qmStp+nwoHMCWei1JMA5uRuYHPmEtZY7nnoFUiDW3tGBAdj5m4HnoQQX1pDAwxs2t1DbbIR5I7NK3rjsoL3rNpzcM8hrfI9rHqrquKw9eMRjcEMu5grz68CLotflDIh136fl5Dd2vwHKjuQoZ69vd4qKqumlL0tDxq4P7ksh6Qi14J5QoR7HmmJ7pqezxnyjItZv7ktStLV4FhsNgkZsnFP7tz6Z4cLIe6zTV3+d95qn8MnDUkBuCgghO4HaCAiAd3fMzBebMsobUgIYWEO1oDfAOk1YMG85zhkAUuNu8YtCgkQexIgjEQrQEvIzOew4AKH1KsfQKb1TxQA-vQ+g5DUg3xXk3RqhsJD0DIECR6Nkc52XzKCAAotJNOMgiBhG4qBXiEFFYqkXIxKEuVXTrWAEQSgUdlFplUewdRIRs4lzzCID0nlzTPVzt0O2kcXIpmiFCeuYIWQASWJebxQNJAaiESIsRv9HFSNsQmDunir7VSZMI0RMBwnEOkSzDYiAiBhTIGQZAvtF41jZASBYtFVCt0kTYj0hZWqIFBDUTUFAiCFKWMUmopSo4wloMoY0iBi6xiOFE0hsJTi1ihPdKyT1UmDMFMKNOYimEezfjAThVEl4UjvgIgW8grH9JtIM3uy52j4LLCPau-Mj6TzYdQlZPsb4bJrp8R+xQW7P3sC2JZsJeATFSKOK0uCVwDwAXyNUO88EIQ3CPfCu4DxULPOROhVF7zEVQDBTiCyWEfJuczO599PiCXRZck808vZUR4Ws-hDzoBCMsuIv+YLB6IRHkKUkcyoSYC+WAFmVphDcH-F6MgWgwKSj8IpZS2BrpkH+Uc8Fw8QXRgqSbByXcLaS23NCo8Vy4W0NnoijVbYUVmTRTXC5J9NXvxJbclK9z+Z4rOdTZhhLgpnwtfPOJ5LrUJXxcah1dNJAcuAL8g4EFcVevOT6z4oq3DirUmQEN8ghHjNpREuMMTlVfRiUQAlpqiVnh7Aiy16zcWtTml1V4NKUkKvbmwj6Kq+pnSUSEOBqz2XfK5QcGCuiPAqn9YG8YEFzLlsmZW+y5cFiV0gKCfWf1AY1BBhDaGQSPDJTYfA4cLbOW9u4OTSm3rFkrubf69m0Qg481xULaAIsxaQHVmEaWstMAKxCErFWvF1aawvTrPW7gDY7JtiOzuY7u6W3ldYqY9x8722pPQJ2LtCq7oxfun267-aBy5qezZ3xPyFyCSqISkB46J1NMnAcqd04yF4lncpoG84FwdlHapxDU2AdrVmz4frvm1VxUbYdTHzarAOf-LeI8gIXtFuLUEiMtK4AIKTQS8hEZXl4OAYA6sRPyHkL2K0oIAImXU5o49CAIRkbAoEjUEnEbxqpWWn+T1BOMvAcIPJYmYD-UBlJ+E+BE3hLsxChzskuBQnY2AWq0AKAWFMk2pDh7N0wQYN9VEUrVwMt873f+BDgWgkwISLUKppMIhhJeJJMgwF4fwCJeErwBhpZOSCi07wECZey403LHnoRkFC2iDERoJDmVeEAA). 

## Key Concepts

- **Concordium ID** – The native ID layer of the Concordium network. Users undergo an identity check based on government-issued identity documents. See more in[ the Concordium docs](https://docs.concordium.com/en/mainnet/docs/protocol/identity.html).
- **Concordium ID App (ID App)** – The Concordium mobile identity wallet that stores user credentials and generates zero-knowledge proofs in response to Verifiable Presentation Request (VPR) (app is available on[ iOS](https://apps.apple.com/in/app/concordium-id-app/id6746754485) and[ Android](https://play.google.com/store/apps/details?id=com.idwallet.app)).
- **Identity Credential** – A digitally signed credential issued by an Identity Provider or on-chain entity containing verified user attributes.
- **Merchant** – an (online) service provider that needs to restrict access to certain features or products.
- **Verification Audit Record (VAR)** – An off-chain log maintained by the Merchant storing the request, response, and metadata for audit purposes.
- **Verifiable Presentation (VP)** – A cryptographically verifiable proof generated by the Concordium ID App showing that certain conditions are met without revealing the user’s private data.
- **Verifiable Presentation Request (VPR)** – A structured request from the Merchant specifying what the user must prove and in what context.
- **Verification Audit Anchor (VAA)** – An on-chain record (a `RegisterData` transaction) anchoring the Verification Audit Record on the blockchain. The VAA does not contain any private data from the VAR.
- **Verification Request Anchor (VRA)** – An on-chain record (a `RegisterData` transaction) anchoring the event that a verification request has been made. The entity responding to the request must link to the VRA (by including the relevant block hash) as part of the proof.
- **RegisterData transaction**: A type of blockchain transaction which adds a small message to the Concordium chain, often a hash value of some data. This information is visible on the chain and cannot be removed. Note that adding this message incurs a transaction fee paid in the native currency, CCD.
- **Zero-Knowledge Proof (ZK-Proof)** – A cryptographic proof that lets a user prove something (e.g., age \> 18) without disclosing other information (e.g., a specific birth date).
- **Concordium Verifier Service **– A self-hosted service that allows merchants to create verifiable presentation request(VPR) and verify verifiable presentation(VP). 

# Setup

## Prerequisites

In order to integrate the **Verify & Access** flow in a system, the following prerequisites must be met:

1. Integrate WalletConnect protocol
2. Install Concordium Web SDK
3. Set up Concordium account with CCD (to cover transaction fees)
4. Docker

## Setup Guide

### **1. Integrate the WalletConnect Protocol**

Merchant app must support the [WalletConnect Protocol](https://reown.com/) for connecting with other wallets and IDApp. WalletConnect is an open protocol that enables secure, decentralized communication between applications (such as a merchant’s frontend) and mobile wallets (such as the Concordium IDApp). It uses end-to-end encryption via a shared session key, allowing both apps to exchange messages over a relay network without requiring direct network access.

Merchant app may use *SignClient*, a lightweight WalletConnect client ideal for Web2 frontends (React, Next.js, Vue, plain JS). *SignClient* is the low-level WalletConnect v2 client SDK that gives developers full control over implementing wallet connectivity without relying on any prebuilt UI components. It enables your application to establish secure sessions with mobile wallets such as the Concordium IDApp, send custom requests like `request_verifiable_presentation_v1`, and listen to important lifecycle events including session updates, authentication events, disconnects, and request expiries. 

Because *SignClient* focuses purely on the protocol layer and does not provide UI widgets, it is ideal for Web2 applications that want to manage their own QR code generation, user flows, and connection logic. For full details, refer to the official documentation: <https://docs.walletconnect.network/> 

#### **Initialize the Client**

```js
import SignClient from "@walletconnect/sign-client";

const wc_client = await SignClient.init({
  projectId: "<YOUR_PROJECT_ID>", // Create your project ID from Wallet connect's developer dashboard
  metadata: {
    name: "Merchant App",
    description: "Merchant connecting to Concordium IDApp",
    url: "https://merchant.com",
    icons: ["https://merchant.com/icon.png"],
  },
});
```

#### **Create a Connection Request**

```js
const { uri, approval } = await wc_client.connect({
  optionalNamespaces: {
      ccd: {     
        methods: ["request_verifiable_presentation_v1"],     
        chains: ["ccd:4221332d34e1694168c2a0c0b3fd0f27", "ccd:9dd9ca4d19e9393877d2c44b70f89acb"], 
      }   
  },
  pairingTopic: undefined,
});
```

To connect with the IDApp, Merchant app should include the Concordium’s [Proposal Namespace](https://specs.walletconnect.com/2.0/specs/clients/sign/namespaces#proposal-namespace)  (as mentioned above code snippet) when initiating a connection via the WalletConnect protocol:

When `uri` is returned, display it to the user as a QR code.

#### **Handle the Wallet Approval**

```js
approval().then((session) => {
  console.log("Wallet connected:", session);
}).catch((err) => {
  console.error("Connection rejected:", err);
});
```

#### **Listening to Important Events**

WalletConnect emits events related to the current session. Read more about events [here](https://docs.reown.com/advanced/api/sign/wallet-usage#setting-up-event-listeners). 

```js
// IDApp fires a session event
wc_client.on("session_event", (event: any) => {})

// User disconnects from IDApp
wc_client.on('session_delete', (data: any) => {})

// User didn’t respond in time
wc_client.on('session_request_expire', (data: any) => {})
```

### **2. Setup Concordium Account in Backend**

To anchor both the **Verification Request Anchor (VRA)** and the **Verification Audit Anchor (VAA)** on-chain, the merchant must submit `RegisterData` transactions to the Concordium blockchain. This requires the merchant to create and maintain a Concordium account (a public-key cryptographic key pair), with its secret key securely stored for signing all audit-related transactions. Since these operations involve submitting on-chain transactions, the account must also be funded with sufficient CCD to cover the required transaction (gas) fees.

#### Preparing a Concordium Account

Merchant developers can create a Concordium account using either of the following official wallets:

- **Browser Wallet:** [Setup Concordium Browser Wallet](https://docs.concordium.com/en/mainnet/docs/browser-wallet/setup-browser-wallet.html)
- **Mobile Wallet:** [Setup Concordium Mobile Wallet](https://docs.concordium.com/en/mainnet/docs/guides/setup-concordium-wallet.html)

Once your account is setup, you can copy the [seed phrase from wallet](https://docs.concordium.com/en/mainnet/docs/browser-wallet/setup-browser-wallet.html#menu) and use it in the backend when creating [Concordium GRPC client in step 2.3.3](https://concordium.atlassian.net/wiki/spaces/CSS/pages/2060320769/Verify+Access+Technical+Implementation+Guide+v0.4#2.3.3-Create-Concordium-Blockchain-gRPC-Client-with-Signer).

The CCD token is publicly available. Read more on [concordium.com/ccd-token](https://www.concordium.com/ccd-token).

### **3. Setup Concordium Verifier Service**

The **Concordium Verifier Service** is a self-hosted service that allows merchants to:

- Create **Verifiable Presentation Requests (VPRs)**
- Verify **Verifiable Presentations (VPs)**

The service is distributed as a Docker container and can be run locally or in production.

Pull the official Concordium verifier service image from Docker Hub:

```bash
docker pull concordium/credential-verification-service:0.1.0
```

Run the container using the following command:

```bash
docker run --rm \
  -e CREDENTIAL_VERIFICATION_SERVICE_NODE_GRPC_ENDPOINT="https://grpc.testnet.concordium.com:20000" \
  -e CREDENTIAL_VERIFICATION_SERVICE_API_ADDRESS="0.0.0.0:8000" \
  -e CREDENTIAL_VERIFICATION_SERVICE_MONITORING_ADDRESS="0.0.0.0:8001" \
  -e LOG_LEVEL="info" \
  -v ./privatekey.export:/keys/verifier_account.export:ro \ 
  -e CREDENTIAL_VERIFICATION_SERVICE_ACCOUNT="/keys/verifier_account.export" \
  -p 8000:8000 \
  -p 8001:8001 \
  concordium/credential-verification-service:0.1.0
```

| Parameter | Description |
| --- | --- |
| `CREDENTIAL_VERIFICATION_SERVICE_NODE_GRPC_ENDPOINT` | Concordium node GRPC endpoint (Testnet in this example) |
| `CREDENTIAL_VERIFICATION_SERVICE_API_ADDRESS` | Address where the verification API is exposed |
| `CREDENTIAL_VERIFICATION_SERVICE_MONITORING_ADDRESS` | Address for health and metrics endpoints |
| `LOG_LEVEL` | Log verbosity (`info`, `debug`, `warn`, `error`) |
| `CREDENTIAL_VERIFICATION_SERVICE_ACCOUNT` | Path **inside the container** to the exported account key |
| `-v ./privatekey.export:/keys/verifier_account.export:ro` | Mounts the exported account key (from Step 2) as read-only |
| `-p 8000:8000` | Exposes the Verification API |
| `-p 8001:8001` | Exposes health and monitoring endpoints |

# Implementation guide

In the below section, this guide will present step-by-step walkthrough of an ID gated access request, with explanations of how the system handles the activity and how to implement this into a given service.

This diagram illustrates how a merchant application integrates with the **Concordium Verifier Service** to request and verify privacy-preserving identity proofs from users via **Concordium ID–supported wallets**, with anchoring performed on the **Concordium blockchain**.

### End to End flow: 

- Merchant UI initiates a verification request.
- Merchant Backend calls the **Concordium Verifier Service**.
- Verifier Service creates and anchors a VPR (VRA on-chain).
- Merchant UI connects to the user’s Concordium ID wallet via WalletConnect.
- Wallet generates a privacy-preserving proof.
- Verification result is returned and processed by the merchant system and using Verifier Service.

Below is a step-by-step, detailed explanation of the complete verification flow, written in a way that helps integrators understand what happens, who is responsible, and why each step exists.

## Step 1: User Scans the QR code / uses deep link on Merchant’s Website To Connect the IDApp

In the Concordium ecosystem, WalletConnect is used to establish a secure communication channel between the merchant’s web frontend and the user’s Concordium IDApp. When the user interacts with the merchant’s website, a WalletConnect URI is generated to initiate a pairing session. Typically, if the IDApp is on a different device, the user scans the QR code shown on the web page, which transfers this URI to the IDApp and completes the pairing.

However, when both the web app and the IDApp are running on the same device, scanning a QR code isn’t required. Instead, the user simply taps a deeplink button, which invokes the IDApp directly and passes the WalletConnect URI internally. Regardless of whether the URI is transferred via QR scan or deeplink, the pairing session that follows is cryptographically secured and can be reused for future interactions—meaning the user doesn’t need to repeat the scanning or deeplink process unless the session expires or is explicitly terminated. WalletConnect v2 sessions typically last about 30 days by default.

During this step:

- The merchant’s frontend generates a WalletConnect URI.
- If the IDApp is on another device, the frontend renders this URI as a QR code, which the user scans with the Concordium IDApp.
- If the IDApp and the web app are on the same device, the frontend instead shows a deeplink button, and tapping it opens the IDApp directly with the WalletConnect URI (no scanning required).
- In both cases, the IDApp receives the URI and initiates the secure WalletConnect pairing handshake.
- The pairing is then approved on both sides, completing the session setup.

## Step 2: Merchant’s Backend Generates the Verifiable Presentation Request (VPR)  

Once the pairing between the **Merchant Website** and the **IDApp** is established via WalletConnect , the merchant sends a **Verifiable Presentation Request (VPR)** to the Concordium IDApp.

The merchant must first create **Verification Request Anchor(VRA) **and then, based on this, generate **VPR**. It defines *what* must be proven (credential statements) and *under what context* (e.g., transaction, session, or page). The VRA is later referenced by the IDApp when generating the VP.

To enable this, merchants call the** Concordium Verifier Service **API (for example, `/verifiable-presentations/create-verification-request`) to generate the  **Verifiable Presentation Request (VPR) **(and internally ), which is returned to their frontend. During this process the **Concordium Verifier Service** also createsthe **Verification Request Anchor (VRA)**.

### **2.1. Merchant Backend API**

The merchant’s backend provides an API endpoint that the frontend can call to create a Verifiable Presentation Request.

```
POST /verifiable-presentations/create-verification-request
Content-Type: application/json

{
   "connectionID": "<WalletConnect connection/session ID>"
}
```

The frontend must include the `connectionID` (i.e., the WalletConnect session ID) in the request body so that it can be embedded into the context of the Verifiable Presentation Request. This ensures the request is bound to the correct WalletConnect session, as described in **Section 2.2.1**.

#### **2.2 Generate a Verifiable Presentation Request using Concordium Verifier Service**

#### **2.2.1 Define Context **

Before generating a verifiable presentation request, the **context** must be defined.  
The context acts as the unique identifier for the proof request, ensuring **replay protection**, **binding to a specific session**, and human-readable traceability during verification and audit.

##### **2.2.2 Define Credential Statement **

The **credential statement** defines the **conditions** that the holder’s Identity Credential must satisfy in order to fulfill the proof request. These statements specify **what attributes** should be proven and **what constraints** they must meet — for example, proving age, nationality, or residency — without revealing unnecessary personal data.

```js
// Prove Age Is Within a Range (Age Verification)
{
  "type": "AttributeInRange",
  "tag": "age",
  "lowerBound": 18,
  "upperBound": 100
}

// Prove Nationality Is One of an Allowed Set
{
  "type": "AttributeInSet",
  "tag": "nationality",
  "set": ["IN", "IE", "US", "UK"]
}

// Prove Residence Is Not in a Restricted Set. e.g the statement below asks if the wallet owner is a citizen of China or North Korea.
{
  "type": "AttributeNotInSet",
  "tag": "residence",
  "set": ["CN", "KP"]
}
```

Read more about different identity proofs available in  <https://docs.concordium.com/en/mainnet/docs/network/guides/create-proofs.html#create-proofs> You can refer to[this documentation](https://docs.concordium.com/en/mainnet/tools/wallet-sdk/wallet-sdk-identity-provider.html) to get the list of supported IDPs and their indexes. 

##### **2.2.3 Generate Verification Request Anchor (VRA) and Verifiable Presentation Request (VPR)**

In this step, the merchant uses API `POST /verifiable-presentations/create-verification-request`  of the **Concordium Verifier Service** to generate a **Verifiable Presentation Request (VPR)** and anchor it on the Concordium blockchain as a **Verification Request Anchor (VRA)**.

The VPR is constructed by combining:

- The verification **context** (connection ID, resource, and optional metadata)
- The **credential statements** defining what must be proven
- The **trusted issuer (IdP) set** whose credentials are accepted

Anchoring creates an immutable, on-chain **VRA** containing the hash of the **context** and **credential statements**. This allows the IdApp and auditors to verify that any Verifiable Presentation (VP) corresponds exactly to the originally anchored request.

The following example generates a VPR requiring proof that the holder is **at least 18 years old**, suitable for a compliant wine shop.

```js
curl -X POST https://verifypay-api-staging.nanocorp.io/api/verifiable-presentations/create-verification-request \
  -H "Content-Type: application/json" \
  -d '{
  "connectionId": "wallet_connect_session_topic",
  "resourceId": "/api/v1/home",
  "contextString": "Age verification for compliant wine shop",
  "publicInfo": {
    "domain": "google.com",
    "name": "Alphabet Inc",
  },
  "requestedClaims": [
    {
      "type": "identity",
      "source": ["identityCredential"],
      "issuers": [
          "did:ccd:testnet:idp:0",
          "did:ccd:testnet:idp:1", 
          "did:ccd:testnet:idp:2", 
          "did:ccd:testnet:idp:3"],
      "statements": [
        {
          "type": "AttributeInRange",
          "attributeTag": "dob",
          "lower": "19000101",
          "upper": "20070101"
        }
      ]
    }
  ]
}'
```

Sample VPR API response: 

```json
{
  "type": "ConcordiumVerificationRequestV1",
  "context": {
    "type": "ConcordiumUnfilledContextInformationV1",
    "given": [
      {
        "label": "Nonce",
        "context": "8997ddddb52f19db0176bc53ac99b622a30b39cc26db7cc15b8fe85157b5469b"
      },
      {
        "label": "ConnectionID",
        "context": "12311123-123123-AAA"
      },
      {
        "label": "ContextString",
        "context": "My compliant web3 wine shop"
      },w
      {
        "label": "ResourceID",
        "context": "api/v1/home"
      },
    ],
    "requested": [
      "BlockHash",
    ]
  },
  "subjectClaims": [
    {
      "type": "identity",
      "source": [
        "identityCredential",
      ],
      "statements": [
        {
          "type": "AttributeInRange",
          "attributeTag": "dob",
          "lower": "18000101",
          "upper": "20071203"
        },
        {
          "type": "AttributeInSet",
          "attributeTag": "countryOfResidence",
          "set": [
            "AT",
            "BE",
            ...
            "SE",
            "HR"
          ]
        },
        {
          "type": "RevealAttribute",
          "attributeTag": "firstName"
        }
      ],
      "issuers": [
        "did:ccd:testnet:idp:0",
        ..
        "did:ccd:testnet:idp:4"
      ]
    }
  ],
  "transactionRef": "24dc34ae08c822be028f91efa06ed068d49f3327624b8e96096e97b8bb2a3cc7"
}
```

## Step 3: Merchant’s Frontend Sends the Verifiable Presentation Request (VPR) to IDApp Through Wallet Connect

Once Merchant’s Fronted fetches a **Verifiable Presentation Request (VPR)** from its backend it delivers it to the *Concordium IDApp* over a secure WalletConnect session.   

## Step 4: The IDApp Verifies the VPR, generates the Verifiable Presentation (VP) and sends to Merchant’s Frontend

Once the user approves the proof request in the Concordium IDApp, the app validates the Verifiable Presentation Request (VPR) received from the merchant by interacting with the Concordium blockchain. As part of this process, the IDApp verifies the on-chain anchor using the transaction hash included in the VPR. It retrieves the corresponding transaction from the blockchain, reads the hash stored in the Verification Audit Anchor (VAA), and independently re-hashes **context** and **credential statements** it received from the merchant in the VPR. If the two hashes match, the IDApp confirms that the request is correctly anchored on-chain.

The IDApp then generates a **Verifiable Presentation (VP)** — a zero-knowledge proof demonstrating the requested claims without exposing private data — and securely sends it back to the *merchant’s frontend* via the established WalletConnect session. 

Sample VP: 

```json
{
  "type": [
    "VerifiablePresentation",
    "ConcordiumVerifiablePresentationV1"
  ],
  "presentationContext": {
    "type": "ConcordiumContextInformationV1",
    "given": [
      {
        "label": "Nonce",
        "context": "c007beb272448c65f07664ba41d3d155cbdafa10541872b3cc69abee6aa56e79"
      },
      {
        "label": "ConnectionID",
        "context": "12311123-123123-AAA"
      },
      {
        "label": "ContextString",
        "context": "My compliant web3 wine shop"
      },
      {
        "label": "ResourceID",
        "context": "api/v1/home"
      },
    ],
    "requested": [
      {
        "label": "BlockHash",
        "context": "b6f8af7da186e5ef34ee706f5daf80caa9041ba721c178ca1db633e04320a1eb"
      }
    ]
  },
  "verifiableCredential": [
    {
      "type": [
        "VerifiableCredential",
        "ConcordiumVerifiableCredentialV1",
        "ConcordiumIdBasedCredential"
      ],
      "credentialSubject": {
        "id": "did:ccd:testnet:encidcred:02000300000001b1fa2345d...39f",
        "statement": [
          {
            "type": "AttributeInRange",
            "attributeTag": "dob",
            "lower": "18000101",
            "upper": "20071118"
          },
          {
            "type": "AttributeInSet",
            "attributeTag": "countryOfResidence",
            "set": [
              "AT",
              ...
              "SI",
              "SK"
            ]
          },
          {
            "type": "AttributeValue",
            "attributeTag": "firstName",
            "attributeValue": "John"
          }
        ]
      },
      "validFrom": "2025-09-01T00:00:00Z",
      "validUntil": "2026-09-30T23:59:59Z",
      "issuer": "did:ccd:testnet:idp:0",
      "proof": {
        "created": "2025-11-17T11:56:17.285Z",
        "proofValue": "000000000000000d...9531",
        "type": "ConcordiumZKProofV4"
      }
    }
  ],
  "proof": {
    "created": "2025-11-17T11:56:17.285Z",
    "proofValue": "",
    "type": "ConcordiumWeakLinkingProofV1"
  }
}
```

## Step 5: The Merchant’s Backend Verifies the VP and Allows access to protected resource

The frontend forwards the** VP** and original **VPR **to the *merchant’s backend* through a `/verifiable-presentations/verify` API call. 

```js
POST /verifiable-presentations/verify
Content-Type: application/json

{
  "presentationRequest": <Verifiable Presentation (VP)>,
  "presentation": <VerifiablePresentationRequest (VPR)>
}
```

The backend uses the **Concordium Verifier Service **to verify the VP cryptographically. Upon successful verification, the merchant grants the user access to the protected resource or completes the requested transaction.

In this step, the merchant uses API `POST verifiable-presentations/verify`  of the **Concordium Verifier Service** to validate the** Verifiable Presentation(VP) **against the** VPR **which:

- Confirms the VP satisfies the statements requested in the VPR.
- Ensures the VP is contextually bound to prevent replay attacks (e.g., checks blockhash, nonce, or resource ID in the context).
- Validates zero-knowledge proofs for ID-based or account-based credentials as specified in the VP.

```js
// 1. Generate unique identifier for the verification event
const uuid: string = crypto.randomUUID(); 

// 2. Make an API call to verifier service to verify the presentation
curl -X POST http://localhost:8000/verifiable-presentations/verify \
  -H "Content-Type: application/json" \
  -d '{
  "auditRecordId": uuid,
  "publicInfo": {
    "key1": "6676616C756531",
    "key2": "6676616C756532"
  },
  "presentation": {
    "presentationContext": {
      "type": "ConcordiumContextInformationV1",
      "given": [
        {
          "label": "Nonce",
          "context": "4f726e383ca2d8d3db...735c359e1d26082fbc8a0b"
        },
        {
          "label": "ConnectionID",
          "context": "b10b78ab4c3a0c3b9...9bff"
        },
        {
          "label": "ContextString",
          "context": "ID Proof Verification for Concordium Plugin"
        },
        {
          "label": "ResourceID",
          "context": "/api/v1/verify/id-proof"
        }
      ],
      "requested": [
        {
          "label": "BlockHash",
          "context": "37c4431ca8cae5b...0bf24c42cf"
        }
      ]
    },
    "proof": {
      "created": "2026-01-05T13:25:08.655487015Z",
      "proofValue": "",
      "type": "ConcordiumWeakLinkingProofV1"
    },
    "type": [
      "VerifiablePresentation",
      "ConcordiumVerifiablePresentationV1"
    ],
    "verifiableCredential": [
      {
        "type": [
          "VerifiableCredential",
          "ConcordiumVerifiableCredentialV1",
          "ConcordiumIdBasedCredential"
        ],
        "credentialSubject": {
          "id": "did:ccd:testnet:encidcred:02000300000001...6",
          "statement": [
            {
              "type": "AttributeInRange",
              "attributeTag": "dob",
              "lower": "18750106",
              "upper": "20080105"
            }
          ]
        },
        "validFrom": "2025-12-01T00:00:00Z",
        "validUntil": "2026-12-31T23:59:59Z",
        "issuer": "did:ccd:testnet:idp:0",
        "proof": {
          "created": "2026-01-05T13:25:08.655487015Z",
          "proofValue": "000000...4",
          "type": "ConcordiumZKProofV4"
        }
      }
    ]
  },
  "verificationRequest": {
    "type": "ConcordiumVerificationRequestV1",
    "context": {
      "type": "ConcordiumUnfilledContextInformationV1",
      "given": [
        {
          "label": "Nonce",
          "context": "4f726e383ca...a0b"
        },
        {
          "label": "ConnectionID",
          "context": "b10b78ab4c...c9bff"
        },
        {
          "label": "ContextString",
          "context": "ID Proof Verification for Concordium Plugin"
        },
        {
          "label": "ResourceID",
          "context": "/api/v1/verify/id-proof"
        }
      ],
      "requested": [
        "BlockHash"
      ]
    },
    "subjectClaims": [
      {
        "type": "identity",
        "source": [
          "identityCredential"
        ],
        "statements": [
          {
            "type": "AttributeInRange",
            "attributeTag": "dob",
            "lower": "18750106",
            "upper": "20080105"
          }
        ],
        "issuers": [
          "did:ccd:testnet:idp:0",
          "did:ccd:testnet:idp:1",
          "did:ccd:testnet:idp:3"
        ]
      }
    ],
    "transactionRef": "2e6d830e011c1d2b57d11f89ef31cb3fd3e4527d5244cc203ed2179db7be0820"
  }
}'
```

The Verifier Service  performs a complete verification of a presentation against a given verification request. It ensures that the request and presentation contexts match, confirms that all involved credentials are active, checks that the request’s anchor is registered on-chain, and finally validates all presentation claims against the request’s statements. After running  inthese checks, the function returns a structured response of the form `{ "result": "verified", "verificationAuditRecord": { ... }, anchorTransactionHash: "presentation_anchor_transaction_hash" }`, where the `result` field indicates whether the verification succeeded or failed. 

Sample API Response:

```json
{
    "result": "verified",
    "verificationAuditRecord": {
        "type": "ConcordiumVerificationAuditRecord",
        "version": 1,
        "id": "recordid1234",
        "request": {
            "type": "ConcordiumVerificationRequestV1",
            ....
        },
        "presentation": {
            "type": [
                "VerifiablePresentation",
                "ConcordiumVerifiablePresentationV1"
            ],
            ....
        }
    },
    "anchorTransactionHash": "72d7dcc421db1db2dcf729c135c40b445b3ddd79a34e8b77f0c605e66317b6e6"
}
```

## Step 6: Merchant’s Backend Creates Audit Record and Stores Private Data in its Database

To ensure transparency, traceability, and regulatory compliance, each successful verification event is captured through an auditable, tamper-evident audit mechanism anchored on the Concordium blockchain. This mechanism consists of two complementary components: the **Verification Audit Anchor (VAA**) and the **Verification Audit Record (VAR)**.

The **VAA** is a minimal, privacy-preserving audit marker registered on the Concordium blockchain by the Verifier Service. It contains only a cryptographic hash of the corresponding VAR, ensuring that no sensitive data is ever placed on-chain. The VAA serves as an immutable timestamp and public reference, proving that a specific verification request occurred at a particular moment while enabling transparent, tamper-proof verification without exposing user information.

Meanwhile, the **VAR**—should be stored securely in the merchant’s private database—contains the complete verification details. This includes the statements requested (e.g., “Is the user 18+?”), what the user successfully proved, the credential used, the derived verifiable presentation, and all associated proof metadata. 

Together, the VAA and VAR form a robust audit trail that ensures compliance and integrity while keeping sensitive verification data strictly off-chain. This process may run in the background after a user successfully completes the Verify-Access flow.

Sample **VAR**:

```js
// response.verificationAuditRecord (VAR)
{
        "type": "ConcordiumVerificationAuditRecord",
        "version": 1,
        "id": "recordid1234",
        "request": {
            "type": "ConcordiumVerificationRequestV1",
            "context": {
                "type": "ConcordiumUnfilledContextInformationV1",
                "given": [
                    {
                        "label": "Nonce",
                        "context": "c5756ace67c1e25f6d71a4b3acd5543fff15c320243e993f484cd8290c31115d"
                    },
                    {
                        "label": "ConnectionID",
                        "context": "conn_8291yuw"
                    },
                    {
                        "label": "ResourceID",
                        "context": "some string for resource"
                    },
                    {
                        "label": "ContextString",
                        "context": "optional context string here"
                    }
                ],
                "requested": [
                    "BlockHash"
                ]
            },
            "subjectClaims": [
                {
                    "type": "identity",
                    "statements": [
                        {
                            "type": "AttributeInRange",
                            "attributeTag": "dob",
                            "lower": "19000101",
                            "upper": "20070101"
                        }
                    ],
                    "issuers": [
                        "did:ccd:testnet:idp:0",
                        "did:ccd:testnet:idp:1",
                        "did:ccd:testnet:idp:2",
                        "did:ccd:testnet:idp:3"
                    ],
                    "source": [
                        "identityCredential"
                    ]
                }
            ],
            "transactionRef": "abf90a3954aa6abf92868775834e3d4fe61125ade3dc7d0e61952b4820220e84"
        },
        "presentation": {
            "type": [
                "VerifiablePresentation",
                "ConcordiumVerifiablePresentationV1"
            ],
            "presentationContext": {
                "type": "ConcordiumContextInformationV1",
                "given": [
                    {
                        "label": "Nonce",
                        "context": "c5756ace67c1e25f6d71a4b3acd5543fff15c320243e993f484cd8290c31115d"
                    },
                    {
                        "label": "ConnectionID",
                        "context": "conn_8291yuw"
                    },
                    {
                        "label": "ResourceID",
                        "context": "some string for resource"
                    },
                    {
                        "label": "ContextString",
                        "context": "optional context string here"
                    }
                ],
                "requested": [
                    {
                        "label": "BlockHash",
                        "context": "e3ffe5aa1f2813e5a825d9cb5b00d40527b7ebd494af8f57c2be0d1907f46cd7"
                    }
                ]
            },
            "verifiableCredential": [
                {
                    "type": [
                        "VerifiableCredential",
                        "ConcordiumVerifiableCredentialV1",
                        "ConcordiumIdBasedCredential"
                    ],
                    "credentialSubject": {
                        "id": "did:ccd:testnet:encidcred:020003000000...cf5",
                        "statement": [
                            {
                                "type": "AttributeInRange",
                                "attributeTag": "dob",
                                "lower": "19000101",
                                "upper": "20070101"
                            }
                        ]
                    },
                    "validFrom": "2025-12-01T00:00:00Z",
                    "validUntil": "2026-12-31T23:59:59Z",
                    "issuer": "did:ccd:testnet:idp:0",
                    "proof": {
                        "created": "2026-01-05T13:01:31.483Z",
                        "proofValue": "000000000000000d0...75d48bb6e4a5",
                        "type": "ConcordiumZKProofV4"
                    }
                }
            ],
            "proof": {
                "created": "2026-01-05T13:01:31.483Z",
                "proofValue": "",
                "type": "ConcordiumWeakLinkingProofV1"
            }
        }
    }
```

Sample **Verification Audit Anchor **reference transaction: 

```json
// response.anchorTransactionHash
{
  "anchorTransactionHash": "392ccd3970a4d12742fd5e7fce18bd8d8a03fd181955e62c1bdd52403ce7a67c"
}
```

## Integrating The Concordium Verification Web UI

This guide provides a comprehensive walkthrough for developers to integrate the **ConcordiumVerificationWebUI SDK**. This SDK enables a seamless Verify & Access flow, allowing your website to request privacy-preserving identity proofs (Verifiable Presentations) from users via the **Concordium ID App** or other supported wallets.

## 1. Installation

First, install the SDK from npm

```bash
npm i @concordium/verification-web-ui
```

Then import the ConcordiumVerificationWebUI SDK and its associated styles into your frontend project.

```js
// Step 1 - Import the package
import { ConcordiumVerificationWebUI } from '@concordium/verification-web-ui';
import '@concordium/verification-web-ui/styles';
```

## 2. Initialization

Initialize the SDK with your project metadata and network configuration. You will need a `projectId` from your **WalletConnect** developer dashboard.

```js
// Step 2 - Initialize the SDK
const sdk = new ConcordiumVerificationWebUI({
    network: "testnet", // Use "mainnet" for production
    projectId: "YOUR_WALLETCONNECT_PROJECT_ID",
    metadata: {
      name: "Merchant App",
      description: "Merchant connecting to Concordium IDApp",
      url: window.location.origin,
      icons: [`${window.location.origin}/favicon.ico`],
    },
});
```

## 3. Rendering the UI

The SDK provides built-in UI modules to handle the connection flow, including landing screens and QR code generation for WalletConnect

```js
// Step 3 - Render UIs like landing screens and QR code UI
await sdk.renderUIModals();
```

## 4. Implementing the Verification Flow

The verification process is event-driven. Your frontend must listen for `@concordium/verification-web-ui-event` to manage the lifecycle of the session and proof request.

```js
// Step 4 - Listen to events 
window.addEventListener("@concordium/verification-web-ui-event", handleSDKEvent);

const handleSDKEvent = async (event) => {
  const { type, data } = event.detail;
  switch (type) {
    case "session_approved":
        // See step 5
        break;
    case "presentation_received":
        // See Step 6
        break;
    case "session_disconnected":
        console.log("Wallet session was disconnected by the user.");
        // Reset UI state, clear user session, etc.
        break;
    case "error":
        console.error("An SDK error occurred:", data);
        // Display a user-friendly error message
        alert("An error occurred during verification. Please try again.");
}
```

## 5. Listen to `session_approved` event to Generate and Send VPR

```js
case "session_approved":
  console.log("Wallet connection approved!", data);
  //// Data format
  // {
  //   type: 'session_approved',
  //   data: {
  //     topic: string,
  //     accounts: string[],
  //     namespaces: Record<string, any>
  //   }
  // }

  // 1. Request a VPR from your backend/verifier service
  // Your backend uses the Concordium Verifier Service to create this request.
  const vpr = await generatePresentationRequestFromBackend(
    "AgeVerification", 
    "testnet", 
    18, 
    "gte",
    data.topic
  );

  // 2. Send the VPR to the wallet via the ConcordiumVerificationWebUI SDK
  // 'data.topic' is the specific WalletConnect session topic
  await sdk.value?.sendPresentationRequest(vpr, data.topic);
  
  console.log("VPR sent to wallet successfully.");
  break;
```

> **Note**: Refer to <https://concordium.atlassian.net/wiki/spaces/CSS/pages/2060320769/Verify+Access+Technical+Implementation+Guide+v0.4#Step-2%3A-Merchant%E2%80%99s-Backend-Generates-the-Verifiable-Presentation-Request-(VPR)>  to generate verifiable presentation request (vpr) from backend. 

## 6. Listen to `presentation_received` event to Verify Presentation (proof)

Once the user reviews the request and signs the proof in their wallet, the SDK receives the Verifiable Presentation (VP) and emits the `presentation_received` event. You must then send this proof to your backend for cryptographic verification.

```js
case "presentation_received":
  console.log("Proof received from wallet:", data);
  //// Data format
  // {
  //   type: 'presentation_received',
  //   data: {
  //     verifiablePresentation: any,
  //     presentationContext: any,
  //     proof: any,
  //     type: any,
  //     verifiableCredential: any
  //   }
  // }

  // 1. Forward the proof (VP) to your backend for verification
  // The backend verifies the ZKP against the Concordium blockchain anchor.
  const isValid = await verifyPresentation(data, "testnet");

  // 2. Handle the verification result
  if (isValid.verified) {
    console.log("Identity verification successful!");
    
    // Show the SDK's built-in success UI
    await sdk.value?.showSuccessState();
    
    // Perform your app logic (e.g., redirect to gated content)
  } else {
    console.error("Proof verification failed.");
  }

  // 3. Clean up the UI
  setTimeout(() => sdk.value?.closeModal(), 2000);
  break;
```

> **Note: **Refer to <https://concordium.atlassian.net/wiki/spaces/CSS/pages/2060320769/Verify+Access+Technical+Implementation+Guide+v0.4#Step-5%3A-The-Merchant%E2%80%99s-Backend-Verifies-the-VP-and-Allows-access-to-protected-resource>  to verify verifiable presentation from backend. 

# Important Links

<https://docs.concordium.com/concordium-node-sdk-js/latest/pages/identity-proofs.html>  

<https://docs.concordium.com/en/mainnet/docs/network/guides/create-proofs.html#asking-a-user-to-reveal-attributes>

[Credential Statement Using IdStatementBuilder](https://github.com/Concordium/concordium-node-sdk-js/blob/3982ac7/examples/nodejs/common/statements.ts#L9)

---

# Version Control

| **Version** | **Remarks** | **Date** |
| --- | --- | --- |
| 0.1 | Initial Draft |  |
| 0.2 | Updates following feedback: - Updated *verifiable* to *verification* for anchors - Removed some sections |  |
| 0.3 | Both pre-requisites elaborated: 1. WalletConnect Integration 2. Concordium account set-up 3. ResourceID moved from requested context to given context |  |
| 0.4 | Added Integration of Concordium Verification Service |  |
|  |  |  |
|  |  |  |

# Approval Status

| **Name** | **Date Approved** |
| --- | --- |
| @Torben Pryds Pedersen |  |
| @Andy Nelson |  |
| @Daniel Tschudi |  |

 
