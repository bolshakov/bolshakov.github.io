---
layout: post
title: "Envelope Encryption: A Guide to Secure Data Storage"
date: 2024-11-22 00:00:00 +0000
excerpt: This article shows how to securely store sensitive data using Envelope Encryption - a two-layer approach that combines unique data encryption keys with centralized key management. Explore implementation patterns, security best practices, and real-world examples using cloud key management services.
---

{% include cover_image.html url="/assets/2024-11-22/envelope-encryption-poster.jpeg" %}

Storing user passwords securely is a well-established practice in software development. We hash passwords instead of storing them as plain text to protect against data breaches and malicious attacks. This allows us to verify user passwords by comparing hash values without ever storing the actual passwords.

While hashing works well for passwords, many applications need to store sensitive data that must be retrieved later - like API credentials for third-party services or payment information. This presents a unique challenge: how do we securely store data that needs to remain both protected and accessible?

This challenge is fundamental to computer security. As Saltzer and Schroeder noted, _"The protection of information in computer systems requires [...] a combination of hardware, software, and procedural safeguards"_ [1]. Modern key management systems build upon these principles while addressing new challenges in cloud computing and distributed systems.

This article demonstrates how to implement envelope encryption - a robust approach that significantly improves the security of sensitive data storage while maintaining its accessibility for legitimate use cases. **Envelope Encryption** uses two types of keys:
* A Data Encryption Key (DEK) - a unique key generated for each piece of sensitive data
* A Key Encryption Key (KEK) - also known as a master key, used to encrypt all DEKs

Let's first examine a common approach developers take: encrypting all sensitive data with a single key stored in the application's configuration.

<figure>
    <img src="/assets/2024-11-22/single-key-encryption.png"/>
    <figcaption>Figure 1 - Single Key Encryption</figcaption>
</figure>

While this approach provides a basic level of security, it has several drawbacks:

1. Single point of failure - if attackers gain access to the encryption key, all user data is compromised simultaneously.
2. Key rotation becomes impractical for large files - changing keys requires re-encrypting large volumes of data, like image files.

Additionally, the encryption key is often stored as plain text in application configuration files, making it accessible throughout the application lifecycle and increasing the risk of exposure.

Envelope encryption offers an elegant solution to these problems using a two-layer encryption approach. Let's take a deeper look at how this approach works.

To address the single point of failure, we could generate a unique data encryption key (DEK) for each data element (like payment information or passport scan). This way, if an attacker compromises one key, they only gain access to that specific piece of data rather than the entire dataset.

<figure>
    <img src="/assets/2024-11-22/data-key-encryption.png"/>
    <figcaption>Figure 2 - Data Key Encryption</figcaption>
</figure>

This approach introduces a new challenge: instead of managing a single key in configuration, we're now storing multiple encryption keys in the database. This could actually increase our attack surface through potential database dumps, backup leaks, or log exposures. The solution? Encrypt the DEKs with a master encryption key - also known as a key encryption key (KEK).

<figure>
    <img src="/assets/2024-11-22/envelope-encryption.png"/>
    <figcaption>Figure 3 - Envelope Encryption</figcaption>
</figure>

While this might seem like we've come full circle, the situation is now significantly improved:

* **Efficient key rotation** - only the small DEKs need to be re-encrypted with a new master key, not the potentially large data files themselves.
* **Improved security** - even if attackers get a database dump, they can't access the data without the master key.
* **Reduced blast radius** - compromising a single DEK only exposes one piece of data.

This two-layer approach combines the security benefits of unique encryption keys with centralized key management. However, this approach still has a critical challenge: securing the master key (KEK) that can decrypt all sensitive data. To address this, let's explore key management systems and how they can strengthen our security architecture.

## Key Management Systems

Hardware Security Modules (HSMs) are specialized physical devices designed to safeguard cryptographic keys. They're used across different scales - from Apple's Secure Enclave in iPhones protecting biometric data to financial institutions securing transactions to IANA's Root Key Signing Ceremony, where HSMs guard the DNS root keys. 

<figure>
    <img src="/assets/2024-11-22/root-keys-signing-ceremony.png"/>
    <figcaption>Figure 4 - Root Key Signing Ceremony</figcaption>
</figure>

At IANA, the ceremony requires multiple trusted representatives to use access cards (stored in physical safes) to operate the HSMs (visible on the ceremonial table in the picture above), ensuring no single person can access these critical keys.

While HSMs provide maximum security through physical isolation, software-based key management systems offer flexibility and scale. Solutions like [HashiCorp Vault] allow organizations to implement robust key management within their infrastructure, while cloud providers ([Amazon KMS], [Google Cloud KMS], [Azure Key Vault]) offer managed services with built-in security controls and compliance certifications (FIPS 140-2, SOC, ISO).

Now when we understand HSMs and KMSs, let's see how they solve our master key storage challenge. Key Management Systems don't just store keys - they can perform cryptographic operations without the keys ever leaving the secure environment. This is crucial for our envelope encryption implementation:

1. We store the master key (KEK) securely within the KMS
2. When we need to decrypt sensitive data, we send the encrypted DEK to the KMS for decryption
3. We use the decrypted DEK to access the sensitive information
4. The DEK remains decrypted only for a short period in memory

This approach ensures our master key never leaves the secure KMS environment, significantly reducing the risk of exposure.

<figure>
    <img src="/assets/2024-11-22/envelope-encryption-with-kms.png"/>
    <figcaption>Figure 5 - Envelope Encryption - Sequence Diagram</figcaption>
</figure>

The picture above shows the sequence diagram of the data decryption process using the encrypted envelope approach.

## Best Practices

While envelope encryption provides a sound security foundation, its effectiveness depends on proper implementation and maintenance. A single oversight in key management or access control could compromise the entire system. Therefore, following security best practices is crucial for maintaining the integrity of your encryption system:

**Key Rotation**. Regular key rotation is essential - even if your keys haven't been compromised, they might have been exposed in ways you haven't detected. Having an established rotation schedule limits the window of opportunity for potential attackers.

* Rotate your master key (KEK) regularly according to your security policies
* Implement automated DEK rotation for compromised or aging keys
* Keep old versions of KEK available to decrypt data encrypted with previous DEKs

Proper **Access Management** is your first line of defense - a well-implemented encryption system can be undermined by poor access management. Remember: your system is only as secure as its weakest access point.

* Follow the principle of least privilege for KMS access
* Use separate KEKs for different environments (development, staging, production)
* Implement strong authentication for KMS access
* Consider implementing multi-party authorization for critical operations

Without proper **Monitoring and Auditing**, you might not even know your system has been compromised. Regular auditing and real-time monitoring provide visibility into your encryption system's health and can alert you to potential security incidents before they escalate.

* Log all encryption/decryption operations
* Monitor for unusual patterns in KMS API usage
* Set up alerts for failed decryption attempts
* Regularly audit KMS access logs

Your DEKs are the keys that directly protect sensitive data - their secure generation, storage, and lifecycle management are critical. A single leaked DEK could expose sensitive data, while a lost DEK could make data irrecoverable.

* Generate DEKs using cryptographically secure random number generators
* Never store DEKs in application logs or configuration
* Clear decrypted DEKs from memory as soon as possible
* Implement secure backup procedures for encrypted DEKs
* Never reuse DEKs across different data elements - each piece of sensitive data should have its own unique DEK

## Practical Considerations

While envelope encryption significantly enhances security, implementing it requires careful consideration of several practical aspects.

**Performance** overhead is unavoidable when implementing envelope encryption - each data access now requires multiple operations and network calls. Understanding these impacts helps in designing systems that balance security and performance needs.

* Each data access requires two decryption operations (DEK and data)
* KMS API calls add network latency
* Consider caching strategies for frequently accessed data while balancing security risks
* Batch operations when possible to reduce API calls

**Cost Management**. Understanding Cloud KMS pricing is crucial for cost-effective implementation. As of 2024, Amazon charges for both key storage and API usage, which can quickly add up in high-traffic systems.

* For example, Amazon KMS customer managed keys cost $1/month per key
* Each API call (encrypt, decrypt, re-encrypt) costs $0.03 per 10,000 requests
* Key rotation automatically happens every year (or on-demand) at no additional cost
* Consider the cost impact of your key rotation strategy and access patterns

This pricing model is one of the main reasons we encrypt DEKs with a master key instead of storing them directly in KMS - storing each DEK as a separate KMS key would quickly become cost-prohibitive as your system scales.

**Disaster Recovery and Backup**. A robust backup strategy is crucial - while data can be re-encrypted if needed, losing access to your encryption keys means permanently losing access to your data. Having a sound disaster recovery plan is not optional.

* Regularly backup encrypted DEKs along with your data
* Keep encrypted backups of master key versions for KEK rotation
* Document and regularly test recovery procedures
* Plan for various failure scenarios:
  * KMS service unavailability
  * Database corruption
  * Region-wide outages
  * Accidental key deletion
* Consider compliance requirements for backup retention and key restoration

The most crucial point: while you can always re-encrypt data with new keys, losing both the DEK and KEK means your encrypted data becomes permanently inaccessible.

## Implementation Example

Let's try implementing Envelope Encryption ourselves using [Amazon KMS]. We'll use Python with the AWS SDK (boto3) and omit the database part for simplicity. Note that in a real-world application, you'd want to use proper database storage and add error handling, but this example demonstrates the core concepts of envelope encryption.

First, you need to generate a new KEK. To do this, go to the [AWS Dashboard] and navigate to "Key Management Service" and press the "Create a key" button. If you have an AWS Command Line client configured, you can also create a new key using the following command:

```bash
aws kms create-key \
  --description "Master key for envelope encryption" \ 
  --profile {your-profile-name}
```  

Make sure to replace `{your-profile-name}` with your profile name in the command above. In the output of the command, you'll see the `KeyId` - this is your master key identifier which we will need later. 

```json 
{
    "KeyMetadata": {
        "AWSAccountId": "XXXXXXXXXXXX",
        "KeyId": "1a1d4879-e27c-4a0d-8fcb-f282943affb7",
        "Arn": "arn:aws:kms:us-east-1:XXXXXXXXXXXX:key/1a1d4879-e27c-4a0d-8fcb-f282943affb7",
        "CreationDate": "2024-11-22T15:26:46.902000+01:00",
        "Enabled": true,
        "Description": "Master key for envelope encryption",
        "KeyUsage": "ENCRYPT_DECRYPT",
        "KeyState": "Enabled",
        "Origin": "AWS_KMS",
        "KeyManager": "CUSTOMER",
        "CustomerMasterKeySpec": "SYMMETRIC_DEFAULT",
        "KeySpec": "SYMMETRIC_DEFAULT",
        "EncryptionAlgorithms": [
            "SYMMETRIC_DEFAULT"
        ],
        "MultiRegion": false
    }
}
```

Now let's implement the encryption and decryption functions in Python. First install the `boto3` and `cryptography` libraries:

```bash
pip install cryptography boto3
```

Here's a simple implementation of the `EnvelopeEncryption` service:

```python
from base64 import b64encode

from cryptography.fernet import Fernet
from dataclasses import dataclass
import boto3

@dataclass
class EncryptedData:
    encrypted_dek: bytes
    encrypted_data: bytes   
    
class EnvelopeEncryption:
    def __init__(self, kms, kms_key_id):
        self.kms = kms
        self.kms_key_id = kms_key_id
        
    def encrypt(self, data: str) -> EncryptedData:
        # Generate a DEK
        dek = Fernet.generate_key()
        
        # Encrypt DEK with KMS (KEK)
        encrypted_dek = self.kms.encrypt(
            KeyId=self.kms_key_id,
            Plaintext=dek
        )['CiphertextBlob']
        
        # Encrypt data locally using DEK
        f = Fernet(dek)
        encrypted_data = f.encrypt(data.encode())
        
        return EncryptedData(
            encrypted_dek=encrypted_dek,
            encrypted_data=encrypted_data,
        )
        
    def decrypt(self, encrypted_data: EncryptedData) -> str:
        # Decrypt DEK using KMS
        dek = self.kms.decrypt(
            CiphertextBlob=encrypted_data.encrypted_dek
        )['Plaintext']
        
        # Decrypt data locally using DEK
        f = Fernet(dek)
        decrypted_data = f.decrypt(encrypted_data.encrypted_data)
        
        return decrypted_data.decode()
```

Now we can check how our encryption and decryption work:

```python
# Use the AWS CLI profile for simplicity
session = boto3.Session(profile_name='{your-profile-name}')

# KeyId we saw earlier when created a new key using AWS CLI 
kek_id = "1a1d4879-e27c-4a0d-8fcb-f282943affb7"

# Initialize with your KMS key ID or alias
encryptor = EnvelopeEncryption(kms=session.client("kms"), kms_key_id=kek_id)

# Encrypt some sensitive data
sensitive_data = "4111-1111-1111-1111"
encrypted_record = encryptor.encrypt(sensitive_data)
```
    
After successful encryption we can inspect the `encrypted_record` data class:

```python 
# Show encrypted data
>>> print(f"Encrypted DEK: {b64encode(encrypted_record.encrypted_dek).decode()}")
Encrypted DEK: AQICAHgTbSP63xarroVejfVgFi2eWiUcZQu/3QUtgnGAwKPJAwGWNlEaCMaBH3XsTYEzlqPpAAAAizCBiAYJKoZIhvcNAQcGoHsweQIBADB0BgkqhkiG9w0BBwEwHgYJYIZIAWUDBAEuMBEEDN5Vp+kkPvqfKjOphwIBEIBH2hSTZzshKbMLwU5ZURkoO/MOsU6TZWAH2WAu4iLb1lrOHopMJAVgsAzbjTPXBN7VaKuuldWvw+sKgRWWLpFPaY7YpQdaadk=

>>>print(f"Encrypted Data: {b64encode(encrypted_record.encrypted_data).decode()}")
Encrypted Data: Z0FBQUFBQm5RSmx0V1JySllRNExsdkRkWVBoNDVaUzhabm5GOTU2dmF3V1pJTU9peEpkWmZsenRHdmJ5czBKZ3plUmx6XzF2UURfa2VkOXRpRkJJT0huN1VGMmFsNTZDM1FUb1JjM2ZkT2U4N1FobVdwVW45MEk9
```

Since, we ensured the data was encrypted, we can proceed with decryption.

```python
>>> decrypted_data = encryptor.decrypt(encrypted_record)
>>> print(f"\nDecrypted Data: {decrypted_data}")
Decrypted Data: 4111-1111-1111-1111
```

Indeed, the code outputs the original sensitive data, confirming that our envelope encryption implementation works correctly.

<div class="box purple">
   ⚠️ Don't forget to delete the keys you created in the AWS KMS console or using the AWS CLI after you finish testing. Leaving unused keys in your account can lead to unnecessary costs.
</div>

## Conclusion

Envelope encryption provides a powerful solution for protecting sensitive data while maintaining its accessibility. Its key benefits include efficient key rotation, reduced blast radius in case of compromise, and simplified key management through KMS integration. The two-layer approach allows organizations to leverage the security of hardware security modules or cloud KMS services while keeping data encryption operations local and cost-effective.

This pattern is particularly useful when:
* Storing sensitive data that needs to be retrieved later (unlike passwords)
* Handling large encrypted files where key rotation could be problematic
* Implementing multi-tenant systems where data isolation is crucial
* Meeting compliance requirements for data protection

To get started with envelope encryption:
1. Choose a KMS solution that fits your scale and security requirements
2. Implement proper key management practices from the beginning
3. Set up monitoring and auditing
4. Document and test your disaster recovery procedures

Remember: while envelope encryption adds complexity to your system, the security benefits far outweigh the implementation overhead for sensitive data protection.

## References
[1] J. H. Saltzer and M. D. Schroeder, "The Protection of Information in Computer Systems," Proceedings of the IEEE, vol. 63, no. 9, pp. 1278-1308, 1975.

[HashiCorp Vault]: https://www.hashicorp.com/products/vault
[Amazon KMS]: https://aws.amazon.com/kms/
[Google Cloud KMS]: https://cloud.google.com/security/products/security-key-management?hl=en
[Azure Key Vault]: https://azure.microsoft.com/en-us/products/key-vault/
[AWS Dashboard]: https://aws.amazon.com
