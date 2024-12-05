---
layout: post
title: "Get blockchain dddress from AWS KMS"
---

# Get blockchain address from AWS KMS

In this post, I am going to show you how to get the blockchain address from AWS KMS.

## AWS KMS

AWS Key Management Service (KMS) is a managed service that makes it easy for you to create and control the encryption keys used to encrypt, decrypt, sign, and verify your data.

If you set it up correctly, you can use AWS KMS to sign blockchain transactions, which are based on the secp256k1 elliptic curve, like Ethereum and any other evm compatible blockchain.

I will share with you the Terraform code in order to setup the AWS KMS key. Once we have the KMS set up, I will show you how to get the public key from the KMS key and how to derive the blockchain address from it.

## Terraform

First, let's create the Terraform code to set up the KMS key.

```hcl
resource "aws_kms_key" "example" {
  description              = "ECC_SECG_P256K1 (secp256k1) asymmetric KMS key for signing and verification of EVM transactions"
  customer_master_key_spec = "ECC_SECG_P256K1"
  key_usage                = "SIGN_VERIFY"
  enable_key_rotation      = false
  policy = jsonencode({
    Version = "2012-10-17"
    Id      = "key-default-1"
    Statement = [
      {
        Sid    = "Enable IAM User Permissions"
        Effect = "Allow"
        Principal = {
          AWS = "<arn of user/role>"
        },
        Action   = "kms:*"
        Resource = "*"
      },
      {
        Sid    = "Allow administration of the key"
        Effect = "Allow"
        Principal = {
          AWS = "<arn of user/role>"
        },
        Action = [
          "kms:Create*",
          "kms:Describe*",
          "kms:Enable*",
          "kms:List*",
          "kms:Put*",
          "kms:Update*",
          "kms:Revoke*",
          "kms:Disable*",
          "kms:Get*",
          "kms:Delete*",
          "kms:ScheduleKeyDeletion",
          "kms:CancelKeyDeletion"
        ],
        Resource = "*"
      },
      {
        Sid    = "Allow use of the key"
        Effect = "Allow"
        Principal = {
          AWS = "<arn of user/role>"
        },
        Action = [
          "kms:Sign",
          "kms:Verify",
          "kms:DescribeKey",
          "kms:GetPublicKey"
        ],
        Resource = "*"
      }
    ]
  })
}
```

THat is just a snippet of the Terraform code. You need to replace the `<arn of user/role>` with the ARN of the user or role that you want to have access to the KMS key.

## Get the public key

Once you have the KMS key set up, you can get the public key from it using the AWS CLI.

```bash
aws kms get-public-key --key-id <key-id> --output text --query PublicKey
```

which will give you the public key Base64 encoded.

Great :) We not have the public key. Time to decode it, since it is DER encoded and
calculate the blockchain address from it.

## Decode the public key

Initialize a new JavaSCript project and install the `asn1.js` and `ethers` packages.

```bash
npm init -y
npm install asn1.js ethers
```

Copy the following code and add your public key to the `pubkeyEncoded` variable.

```javascript
const asn1 = require('asn1.js');
const { ethers } = require("ethers");

const pubkeyEncoded = "<Base64EncodedPubKye>";
const publicKeyDer = Buffer.from(pubkeyEncoded, 'base64');

// Define the ASN.1 structure for EC public keys
const ECKey = asn1.define('ECKey', function () {
    this.seq().obj(
        this.key('algorithm').seq().obj(
            this.key('algorithm').objid(),      // e.g., 1.2.840.10045.2.1 (EC Public Key)
            this.key('parameters').objid()     // e.g., 1.3.132.0.10 (secp256k1)
        ),
        this.key('publicKey').bitstr()         // Raw public key in ASN.1 BIT STRING
    );
});

// Parse the DER-encoded public key
const decoded = ECKey.decode(publicKeyDer, 'der');
const publicKey = decoded.publicKey.data; // Raw public key bytes

// Validate length of the public key
if (publicKey.length !== 65) {
    throw new Error(`Invalid public key length: ${publicKey.length}. Expected 64 bytes.`);
}

const publicKeyHex = '0x' + publicKey.toString('hex');

console.log(ethers.computeAddress(publicKeyHex));
```

Run the script and you will get the blockchain address.

```bash
node decodePublicKey.js
```

That's it! You have successfully derived the blockchain address from the AWS KMS public key.

## Conclusion

In this post, I showed you how to get the blockchain address from an AWS KMS public key. This can be useful if you want to use AWS KMS to sign blockchain transactions. But we haven't covered the part on how to submit a transaction to KMS to sign it and forward to a blockchain - maybe in the future.

I hope you found this post helpful. If you have any questions or feedback,
feel free to reach out to me on [Twitter](https://twitter.com/Jacekv).