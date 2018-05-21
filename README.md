# Virgil Pythia .NET/C# SDK

[![Nuget package](https://img.shields.io/nuget/v/Virgil.Pythia.svg)](https://www.nuget.org/packages/Virgil.Pythia/0.1.0-beta) [![GitHub license](https://img.shields.io/badge/license-BSD%203--Clause-blue.svg)](https://github.com/VirgilSecurity/virgil/blob/master/LICENSE)


[Introduction](#introduction) | [SDK Features](#sdk-features) | [Install and configure SDK](#install-and-configure-sdk) | [Usage Examples](#usage-examples) | [Docs](#docs) | [Support](#support)

## Introduction

<a href="https://developer.virgilsecurity.com/docs"><img width="230px" src="https://cdn.virgilsecurity.com/assets/images/github/logos/virgil-logo-red.png" align="left" hspace="10" vspace="6"></a>[Virgil Security](https://virgilsecurity.com) provides an SDK which allows you to communicate with Virgil Pythia Service and implement Pythia protocol for the following use cases: 
- **Breach-proof password**. Pythia is a technology that gives you a new, more secure mechanism that "breach-proofs" user passwords and lessens the security risks associated with weak passwords by providing cryptographic leverage for the defender (by eliminating offline password cracking attacks), detection for online attacks, and key rotation to recover from stolen password databases.
- **BrainKey**. User's Private Key which is based on user's password. BrainKey can be easily restored and is resistant to online and offline attacks.

In both cases you get the mechanism which assures you that neither Virgil nor attackers know anything about user's password.

## SDK Features
- communicate with Virgil Pythia Service
- manage your Pythia application credentials
- create, verify and update user's breach-proof password
- generate user's BrainKey
- use [Virgil Crypto Pythia library][_virgil_crypto_pythia]

## Install and configure SDK

The Virgil .NET Pythia is provided as a package named `Virgil.Pythia`. The package is distributed via [NuGet package](https://docs.microsoft.com/en-us/nuget/quickstart/use-a-package) management system.

The package is available for .NET Framework 4.5 and newer.

**Supported platforms**:
- .Net Core 2.0 (MacOS, Linux)

### Install SDK Package

Installing the package using Package Manager Console:

```bash
Run PM> Install-Package Virgil.Pythia -Version 0.1.0-beta
```

### Configure SDK

When you create a Pythia Application on the [Virgil Dashboard][_dashboard] you will receive Application credentials including: Proof Key and App ID. Specify your Pythia Application and Virgil account credentials in a Pythia SDK class instance.
These credentials are used for the following purposes:
- generating a JWT token that is used for authorization on the Virgil Services
- creating a user's breach-proof password

Here is an example of how to specify your credentials SDK class instance:
```cs
// here set your Virgil Account and Pythia Application credentials
var config = new PythiaProtocolConfig
{
    AppId     = "APP_ID",
    ApiKeyId  = "API_KEY_ID",
    ApiKey    = "API_KEY",
    ProofKeys = new[] {
        "PK.1.PROOF_KEY"
    }
};

var pythia = PythiaProtocol.Initialize(config);
```


## Usage Examples

Virgil Pythia SDK lets you easily perform all the necessary operations to create, verify and update user's breach-proof password without requiring any additional actions and use Virgil Crypto library.

First of all, you need to set up your database to store users' breach-proof passwords. Create additional columns in your database for storing the following parameters:
<table class="params">
<thead>
		<tr>
			<th>Parameters</th>
			<th>Type</th>
			<th>Size (bytes)</th>
			<th>Description</th>
		</tr>
</thead>

<tbody>
<tr>
	<td>salt</td>
	<td>blob</td>
	<td>32</td>
	<td> Unique random string that is generated by Pythia SDK for each user</td>
</tr>

<tr>
	<td>deblindedPassword</td>
	<td>blob </td>
	<td>384 </td>
	<td>user's breach-proof password</td>
</tr>

<tr>
	<td>version</td>
	<td>int </td>
	<td>4 </td>
	<td>Version of your Pythia Application credentials. This parameter has the same value for all users unless you generate new Pythia credentials on Virgil Dashboard</td>
</tr>

</tbody>
</table>

Now we can start creating breach-proof passwords for users. Depending on the situation, you will use one of the following Pythia SDK functions:
- `CreateBreachProofPassword` is used to create a user's breach-proof password on your Application Server.
- `VerifyBreachProofPassword` is used to verify a user's breach-proof password.

### Breach-Proof Password

#### Create Breach-Proof Password

Use this flow to create a new breach-proof password for a user.

> Remember, if you already have a database with user passwords, you don't have to wait until a user logs in into your system to implement Pythia. You can go through your database and create breach-proof user passwords at any time.

So, in order to create a user's breach-proof password for a new database or available one, go through the following operations:
- Take a user's password (or its hash or whatever you use) and pass it into a `CreateBreachProofPassword` function in SDK.
- Pythia SDK will blind a password, send a request to Pythia Service to get a transformed blinded password and de-blind the transformed blinded password into a user's deblinded password (breach-proof password).

```cs
// create a new Breach-proof password using user's password or its hash
var pwd = await pythia.CreateBreachProofPasswordAsync("USER_PASSWORD");

// save Breach-proof password parameters into your users DB
```

After performing `CreateBreachProofPassword` function you get previously mentioned parameters (`Salt`, `deblindedPassword`, `version`), save these parameters into corresponding columns in your database.

Check that you updated all database records and delete the now unnecessary column where user passwords were previously stored.

#### Verify Breach-Proof Password

Use this flow when a user already has his or her own breach-proof password in your database. You will have to pass his or her password into an `VerifyBreachProofPassword` function:

```cs
// get user's Breach-proof password parameters from your users DB

...
// calculate user's Breach-proof password parameters
// compare these parameters with parameters from your DB
var isValid = pythia.VerifyBreachProofPasswordAsync("USER_PASSWORD", pwd);

if (!isValid) 
{
    throw new Exception("Authentication failed");
}
```

The difference between the `VerifyBreachProofPassword` and `CreateBreachProofPassword` functions is that the verification of Pythia Service is optional in `VerifyBreachProofPassword` function, which allows you to achieve maximum performance when processing data. You can turn on a proof step in `VerifyBreachProofPassword` function if you have any suspicions that a user or Pythia Service were compromised.

#### Update breach-proof passwords

This step will allow you to use an `updateToken` in order to update users' breach-proof passwords in your database.

> Use this flow only if your database was COMPROMISED.

How it works:
- Access your Virgil Dashboard and press the "My Database Was Compromised" button.
- Pythia Service generates a special updateToken and new Proof Key.
- You then specify new Pythia Application credentials in the Pythia SDK on your Server side.
- Then you use `UpdateBreachProofPassword` function to create new breach-proof passwords for your users.
- Finally, you save the new breach-proof passwords into your database.

Here is an example of using the `UpdateBreachProofPassword` function:
```cs
// get previous user's VerifyBreachProofPassword parameters from a compromised DB

...

// set up an updateToken that you got on the Virgil Dashboard
// update previous user's Breach-proof password, and save new one into your DB

var updatedPwd = pythia.UpdateBreachProofPassword("UT.1.2.UPDATE_TOKEN", pwd);
```

### BrainKey




## Docs
Virgil Security has a powerful set of APIs, and the documentation below can get you started today.

* [Breach-Proof Password][_pythia_use_case] Use Case
* [The Pythia PRF Service](https://eprint.iacr.org/2015/644.pdf) - foundation principles of the protocol
* [Virgil Security Documenation][_documentation]

## License

This library is released under the [3-clause BSD License](LICENSE.md).

## Support
Our developer support team is here to help you. Find out more information on our [Help Center](https://help.virgilsecurity.com/).

You can find us on [Twitter](https://twitter.com/VirgilSecurity) or send us email support@VirgilSecurity.com.

Also, get extra help from our support team on [Slack](https://virgilsecurity.slack.com/join/shared_invite/enQtMjg4MDE4ODM3ODA4LTc2OWQwOTQ3YjNhNTQ0ZjJiZDc2NjkzYjYxNTI0YzhmNTY2ZDliMGJjYWQ5YmZiOGU5ZWEzNmJiMWZhYWVmYTM).

[_virgil_crypto_pythia]: https://github.com/VirgilSecurity/pythia
[_pythia_use_case]: https://developer.virgilsecurity.com/docs/cs/use-cases/v5/breach-proof-password
[_documentation]: https://developer.virgilsecurity.com/
[_dashboard]: https://dashboard.virgilsecurity.com/
