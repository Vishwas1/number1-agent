# Concordium IDApp SDK v1\.2 Integration Guide

# Introduction 

**Concordium IDApp SDK** — a TypeScript based SDK built for wallet providers who want to easily bring Concordium blockchain and identity features into their apps. With simple, intuitive APIs, this SDK makes integration fast, smooth, and hassle-free — so your wallet can go Concordium-ready in no time.

[Accounts and identities are strongly linked](https://docs.concordium.com/en/mainnet/docs/protocol/manage-accounts.html) on the Concordium platform. To separate concerns, the Concordium ID App can handle the identity-related processes, which allows third-party wallets to simplify their Concordium integration (and focus on signing crypto transactions). This technical specification is intended for third-party wallet developers and focuses on high-level guidance for integrating Concordium account creation and recovery into their wallets. 

**Note: **For clarity, we will refer to these third-party wallets as **Account Wallets** throughout this document, to maintain a clear distinction between account management and identity provisioning.

# Understanding the Concordium blockchain platform

Before diving into implementation details, it’s important to understand the fundamentals of the Concordium blockchain and its account creation process.

- **Concordium Blockchain: **Concordium is a public and permissionless blockchain platform designed for both business applications and public use. It combines privacy-preserving features with regulatory compliance, fast transaction processing and cost-effective operations.
- **Concordium Identity Layer:  **Concordium provides a new solution to blockchain privacy through its identity layer. When users create an account, they first verify their identity with an authorized [identity provider](https://docs.concordium.com/en/mainnet/docs/resources/glossary.html#term-Identity-Provider). On the blockchain, their transactions remain private, but their [identity can be disclosed through a due legal process](https://docs.concordium.com/en/mainnet/docs/snippets/disclosing-identity.html).
- **Concordium Account: **To be able to hold, send, or receive CCD or Protocol Level Tokens (PLT), a user must have account address on the Concordium blockchain. Accounts and [identities](https://docs.concordium.com/en/mainnet/docs/resources/glossary.html#term-Identity) are *strongly linked* on the Concordium Platform. To create an account, user must have a user identity certificate issued by an authorized [identity provider](https://docs.concordium.com/en/mainnet/docs/resources/glossary.html#term-Identity-Provider) on the platform to make sure users are compliant from day one. 
- **Concordium ID App: **A standalone mobile application that allows users to create and manage their identities through supported identity providers. The ID App simplifies the integration process by offloading all identity-related responsibilities from wallet developers, enabling them to focus solely on account management—which is their primary area of expertise—without having to handle the complexities of identity verification and issuance.

# Account creation user journey

The account creation flow begins when a user initiates the process to generate a new Concordium account address through an Account Wallet. The user enables the Concordium network by clicking a toggle button, prompting the IDApp SDK to display a deep link or QR code. Choosing the relevant option initiates a connection with the Concordium IDApp, passing the user’s public key.

**Note:** If the account wallet and IDApp are on different devices, the user chooses that option, which opens a QR code for the user to scan, and takes them to the IDApp

User journey - Account Creation Flow

If the IDApp is already set up and contains an ID credential, the user has the option to either choose the existing ID credential or create a new one by going through the identity verification process with the identity provider (IDP). Once the user selects the ID credential to use, a zero-knowledge proof of identity is generated, which is then used to create a `CreateAccountCredential` transaction. 

The user signs this transaction using the Account Wallet, and the IDApp SDK submits it to the Concordium blockchain. Once the transaction is confirmed, the Concordium account is successfully created, enabling the user to send or receive CCD.

## How to integrate Concordium blockchain to create account

### **Pre-requisite**

*Reown Protocol: *

Account Wallet must support the [ReOwn Protocol](https://reown.com/) (formerly, WalletConnect) for connecting with other wallets and dApps. This same protocol will be used to establish sessions with the IDApp.

**Note:** It is not the responsibility of the IDApp SDK to manage WalletConnect sessions or connections. Account Wallets are expected to handle session initiation, maintenance, and termination.

To connect with the IDApp, Account Wallets should include the Concordium [Proposal Namespace](https://specs.walletconnect.com/2.0/specs/clients/sign/namespaces#proposal-namespace)when initiating a connection via the WalletConnect protocol:

```js
{   
  ccd: {     
    methods: ["create_account"],     
    chains: [ConcordiumIDAppSDK.chainId.Mainnet, ConcordiumIDAppSDK.chainId.Testnet], 
  }
}
```

*Account Tracking in the Wallet:*

It is assumed that the Account Wallet maintains a record of user-created accounts across supported networks, especially for networks where accounts are not enabled by default and require explicit user activation. This includes tracking the associated public key, account address, network, and account index.

### **Account wallet****installs IDApp SDK**

The Concordium IDApp SDK is available as a TypeScript package via NPM. To install it, follow these steps:

1. Install using `yarn` or `npm`

```bash
yarn add @concordium/id-app-sdk
# or
npm install @concordium/id-app-sdk
```

1. You are ready to import the SDK in your project

```js
import {
  ConcordiumIDAppSDK,
  type CreateAccountCreationRequestMessage,
  IDAppSdkWallectConnectMethods,
  type CreateAccountCreationResponse,
  type CreateAccountResponseMsgType,
  type SignedCredentialDeploymentTransaction,
  type RecoverAccountCreationRequestMessage,
  type RecoverAccountResponse,
  type RecoverAccountMsgType,
} from "@concordium/id-app-sdk";
```

### **Account Wallet generates seed and key pairs **

The Account Wallet must generate an **ed25519** key pair (publicKey/signingKey). This can be done using its existing key generation methods. We recommend the account wallet to use the`generateAccountWithSeedPhrase()`provided by the IDApp SDK, which derives the key pair from a seed phrase:

- Using [SLIP-0010](https://github.com/satoshilabs/slips/blob/master/slip-0010.md), similar to the BIP-32/39 key derivation.
- Using the following **derivation path** format: `m/44’/919’/0’/0’/accountIndex’`

```js
const account:CCDAccountKeyPair = ConcordiumIDAppSDK.generateAccountWithSeedPhrase(seed: string, network: Network, accountIndex: number = 0)
```

 The account is of type `CCDAccountKeyPair`

```js
export interface CCDAccountKeyPair {
    publicKey: string;
    signingKey: string;
}
export type Network = 'Testnet' | 'Mainnet';
```

`accountIndex` should be `0` initially and then sequentially incremented to generate new public\_key/signing\_key pair from same seed.  

**Account wallet prepares new account creation** **request** 

Account wallet calls `getCreateAccountCreationRequest()` of the IdApp SDK to create appropriate connection request which accepts two parameters: 

1. `publicKey`: Public key of the user
2.  `reason` : \[Optional\] Provide an appropriate reason of this request

```js
// Prepare the request
const new_account_request: CreateAccountCreationRequestMessage = 
    ConcordiumIDAppSDK.getCreateAccountCreationRequest(public_key,"I want to create a new Concordium account");
```

`CreateAccountCreationRequestMessage` format is as follow: 

```js
export interface CreateAccountCreationRequestMessage {
    publicKey: string;
    reason: string;
}
```

**Account wallet sends the  account creation** **request to IdApp**

Once the `CreateAccountCreationRequestMessage` is formed, account wallet can then make a wallet connect request to IdApp by passing as message. 

```js
const chainId = ConcordiumIDAppSDK.chainId.Mainnet // Or, ConcordiumIDAppSDK.chainId.Testnet
const create_acc_resp: CreateAccountCreationResponse =  await this.wc_client.request({
    topic: this.session.topic,
    chainId,
    request: {
      method,
      params:{ message: new_account_request } ,
    },
});
```

The IdApp processes this request and responds with a `CreateAccountCreationResponse` object.

```js
export interface IDAppError {
    code: IDAppErrorCode;
    details?: string;
}

export declare enum Status {
    SUCCESS = "success",
    ERROR = "error"
}

export interface CreateAccountResponseMsgType {
    serializedCredentialDeploymentTransaction: SerializedCredentialDeploymentDetails;
    accountAddress: string;
}

// Response type you get from IdApp when creating an account
export interface CreateAccountCreationResponse {
    status: Status;
    message: CreateAccountResponseMsgType | IDAppError;
}
```

The `CreateAccountCreationResponse` has following values: 

1. `status`: Indicates whether the account creation was successful ("Success") or not ("Fail").
2. `message`: 
    1. If success, contains details like `CreateAccountResponseMsgType.accountAddress`,`CreateAccountResponseMsgType.serializedCredentialDeploymentTransaction` 
    2. If error, contains details like `IDAppError.code` and `IDAppError.details` 

`serializedCredentialDeploymentTransaction` : A serialised **CreateAccountCredentialTx** unsigned transaction that must be signed and submitted to the Concordium blockchain to activate the account.

> Please note if `chainId` from Account Wallet is different than `chainId`  (selected network) set in IdApp, then the IdApp *switches the network*automatically and let user use ID(s) on the requested network. 

**Note:**  If `chainId` from Account Wallet is different than `chainId` (selected network) set in IdApp, then the IdApp switches the network automatically on the requested network and let user use ID(s) on that network. 

**Account wallet submits the CreateAccountCredentialTx on the blockchain **

The most critical value in the response is `serializedCredentialDeploymentTransaction`, an encoded transaction that proves ownership of the identity. This transaction must be signed and submitted to the Concordium blockchain.   
**Generate signature: **

```js
const signedCredentialDeploymentTx:SignedCredentialDeploymentTransaction = await ConcordiumIDAppSDK.signCredentialTransaction(createAccountCredentialTx, secret_key);
```

The response of `signCredentialTransaction()` if of type `SignedCredentialDeploymentTransaction`

```js
export type SignedCredentialDeploymentTransaction = {
    credentialDeploymentTransaction: CredentialDeploymentTransaction;
    signature: HexString;
}
```

Which has **CreateAccountCredentialTx** and a signature string. 

**Submit the transaction: **

```js
const txHash: string = await ConcordiumIDAppSDK.submitCCDTransaction(signedCredentialDeploymentTx.credentialDeploymentTransaction, signedCredentialDeploymentTx.signature, 'Testnet')
```

> **Notes:**
> 
> - *The *`accountIndex`* index must be incremented after each request, regardless of transaction success or failure.*
> - *The Account Wallet should prompt the user to sign the **CreateAccountCredentialTx**.*
> - *This is a **gasless transaction**, so there is no need to worry about transaction fees.*

**Account Wallet stores keys and newly generated wallet address**

Once the transaction is submitted, it is the responsibility of the Account Wallet to securely store the following:

- The **key pair** (sk, pk)
- The associated **Concordium account address**
- The `accountIndex` index counter

# Account recovery user journey

The Concordium account recovery flow begins when a user wishes to restore access to an existing Concordium account within their account wallet. The user first completes the wallet’s standard recovery process. After recovery, the account wallet can then recover wallet address using IdApp SDK by passing user’s public key.

The account wallet then retrieves the latest transaction sequence number for that account from the blockchain and stores all necessary information such as the account address and sequence number. With the account data fully restored, the user regains access and can now send or receive CCD tokens.

## How to integrate Concordium blockchain to recover an account

The account recovery process is straightforward too. 

**Account wallet retrieves  user’s  public key**

The Account Wallet starts by retrieving the public key of the account to be recovered. If the key was originally generated using the IDApp SDK, this process involves recovering the seed phrase and `account_index`, then deriving the public key using `generateAccountWithSeedPhrase()` of the IDApp SDK.

**Note:** The Account Wallet is expected to manage `account_index` counter, so that it knows how many public\_key/signing\_key pairs have been created from a given seed phrase. 

**Account Wallet recovers the account** 

The Account Wallet then calls the `getKeyAccounts()` method provided by the IdApp SDK. This method connects with Concordium blockchain and looks for associated accounts with the user’s public key of type `KeyAccount`:

```js
const keyAccounts: Array<KeyAccount> = 
        ConcordiumIDAppSDK.getKeyAccounts(public_key, network);
```

`KeyAccount` format: 

```js

export interface KeyAccountPublicKey {
  schemeId: string;
  verifyKey: string; // public key
}

export interface KeyAccount {
  address: string;  // wallet address 
  credential_index: number; 
  is_simple_account: boolean;
  public_key: KeyAccountPublicKey;
}
```

**Note: **In future versions, there will be support for batch recovery of account addresses. 

**Account Wallet stores keys and recovered account address**

After a successful recovery, the Account Wallet must securely store the following:

- The **key pair** (sk, pk)
- The associated **Concordium account address**
- The `accountIndex` index counter

**Account Wallet can query account status from the blockchain **

Account Wallet may use [getAccountInfo()](https://github.com/Concordium/concordium-node-sdk-js/blob/b2f2500eba643215594a8da74862a3f11c0420b8/docs/pages/misc-pages/grpc-v1.md#getaccountinfo) of Concordium [Web-SDK](https://www.npmjs.com/package/@concordium/web-sdk)  to retrieve information about an account from Concordium blockchain by providing `accountAddress` as parameter.  

## Error Handling 

In the event of a failure, the IDApp responds with a status of `Error`, along with a specific error code and corresponding details to help identify the cause.

The available error codes are defined in the `IDAppErrorCode` enum as follows:

```js
export enum IDAppErrorCode {
  AccountNotFound = 1,                 // The specified account does not exist
  AccountCreationFailed = 2,           // Failed to create the account
  DuplicateAccountCreationRequest = 7, // An account with the publicKey already exist
  UnknownError = 99,                   // An unexpected or unknown error occurred
}
```

## IDAppSDK Popups

The `ConcordiumIDAppPoup` module from the `id-app-sdk` enables account wallets to display optional popups for interacting with the Concordium ID App. These popups provide a smooth and embedded user experience for flows like opening the ID App, creating an account, or recovering an account — directly within the wallet interface. To use these popups, wallets must define handler functions (`onIdAppPopup`, `onCreateAccount`) which encapsulate the logic for each action.

### Open IDApp via Deeplink Popup

To open the ID App through a deeplink popup, use the `invokeIdAppDeepLinkPopup` method. 

```js
import { ConcordiumIDAppPoup } from "@concordium/id-app-sdk";

ConcordiumIDAppPoup.invokeIdAppDeepLinkPopup(
  walletConnectUri: "Your wallet connect URI here"
);
```

This popup can be embedded behind a button in the UI, such as:

```html
<button onClick={() => ConcordiumIDAppPoup.invokeIdAppDeepLinkPopup()}>
  Open ID App
</button>
```

### Open IDApp Actions Popup (Account Creation)

To support account creation via the popup, use the `invokeIdAppActionsPopup` method. Provide  `onCreateAccount`  handler to define what should happen in each case.

```js
import { ConcordiumIDAppPoup } from "@concordium/id-app-sdk";

ConcordiumIDAppPoup.invokeIdAppActionsPopup({
  onCreateAccount,
  walletConnectSessionTopic
})
```

**Secure Authentication**

The account creation flow uses a secure authentication mechanism between the user’s Account Wallet and the ID App to guard against identity theft, ensuring that requests can only be delivered to the legitimate Account Wallet rather than a malicious third‐party.  

When the Account Wallet triggers the “Create Account” or “Generate Proof” action via `invokeIdAppActionsPopup()`, the Account Wallet must include the `walletConnectSessionTopic` — a unique session identifier negotiated over WalletConnect—to validate and bind that request to the correct session.

> **Note:** These popup integrations are **optional**. Wallets that choose to use them must implement the handler functions with the desired logic.

---

## Verify Access Usecase

### Identity Proof Pairing Flow

When an Account Wallet needs to facilitate a "Verify & Access" request (e.g., age or residency verification), it can invoke the IDApp deep link popup specifically for proof generation. This is particularly useful for the initial connection between the Account Wallet and the IDApp during a verification session

```js
/**
 * Invokes the IDApp via deep link for Verify-Access flows.
 * This should be used when the Account Wallet needs to connect 
 * to the IDApp to generate a ZK-Proof for a merchant.
 */
ConcordiumIDAppPoup.invokeIdAppDeepLinkPopup({
      // The URI generated by the merchant's WalletConnect session
      walletConnectUri: "wc:your-wallet-connect-uri-here...", 
      
      // Explicitly set for Verify & Access proof generation
      actionType: "generate-proof", 
});
```

### Identity Proof Flow

When a Merchant sends a `request_verifiable_presentation_v1` via WalletConnect, the Account Wallet should use the IDApp SDK to facilitate the proof generation. 

The `invokeIdAppActionsPopup` method should be used to provide a consistent UI for the user. By passing the `onGenerateProof` handler, the Account Wallet can trigger the identity-proving process within the IDApp.

```js
import { ConcordiumIDAppPoup } from "@concordium/id-app-sdk";

/**
 * Trigger the IDApp Action Popup for Verify & Access.
 * This should be called when the Account Wallet receives a 
 * Verifiable Presentation request from a merchant.
 */
ConcordiumIDAppPoup.invokeIdAppActionsPopup({
    // Logic to execute when the user clicks "Generate Proof"
    onGenerateProof: async () => {
        // Implementation logic:
        // 1. Forward the Merchant's proof request to the IDApp 
        // 2. Utilize the 'generate-proof' deep link action
        console.log("User initiated proof generation.");
    },
    
    // The active WalletConnect session topic between the Wallet and IDApp
    walletConnectSessionTopic: "your_session_topic_here"
});
```


## Version Control

| **Version** | **Remarks** | **Date** |
| --- | --- | --- |
| 0.1 | Initial Draft - shared with Coin98 |  |
| 0.2 | - Feedback from stakeholders incorporated - Aligned to PoC version of the SDK |  |
| 0.3 | - Updates to WallectConnect implementation - Account Tracking assumption added |  |
| 0.4 | - Account wallet authentication solution updated |  |
| 1.0 | - Updated installation process in light of making SDK repo public |  |
| 1.1 | - Updated handling of network mismatch |  |
| 1.2 | - Add getAccountInfo method, removed the old recovery flow |  |

## Approval Status

| **Name** | **Date Approved** |
| --- | --- |
| * [ ] @Torben Pryds Pedersen | Assuming the pending comment on error response is resolved as discussed in the comment. |
| * [ ] @Former user (Deleted) | Reviewed with small edits on  . Approved, although there’s still some unresolved comments. |
| * [ ] @Andy Nelson |  |
| * [ ] @Former user (Deleted) |  |

 
