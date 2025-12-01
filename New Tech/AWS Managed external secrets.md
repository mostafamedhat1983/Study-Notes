---
tags:
  - AWS
---
# Using AWS Secrets Manager managed external secrets to manage Third Party secrets

Managed external secrets is a new secret type in AWS Secrets Manager that enables you to store and automatically rotate credentials from integration partners. This feature eliminates the need to create and maintain custom AWS Lambda functions for rotating integration partner secrets. For a complete list of all onboarded partners see [Integration Partners](https://docs.aws.amazon.com/secretsmanager/latest/userguide/mes-partners.html).

When you build applications on AWS, your workloads often need to interact with third-party applications through secure credentials such as API keys, OAuth tokens, or credential pairs. Previously, you had to develop custom approaches to secure and manage these credentials, including building complex rotation Lambda functions that were unique to each application and required ongoing maintenance.

Managed external secrets provides a standardized approach for storing third-party credentials in a predefined format prescribed by each partner. The feature includes automatic rotation that is enabled (by default on the console) during secret creation, complete transparency and user controls for secret management workflows, and the full feature set offered by Secrets Manager including fine-grained permissions management, observability, governance, compliance, disaster recovery, and monitoring controls.

## Key features

Managed external secrets offers several key capabilities that simplify third-party credential management:

- **Lambda-free managed rotation** eliminates the overhead of creating and managing custom rotation functions. When you create an external, rotation is automatically enabled with no Lambda functions deployed in your account.
    
- **Predefined secret formats** ensure that secrets can be properly associated with the integration partner and include the metadata needed for rotation. Each partner defines the required format.
    
- **Integrated partner ecosystem** provides support for multiple partners through a standardized onboarding process. Partners integrate directly with Secrets Manager to offer programmatic guidance for secret creation and managed rotation capabilities.
    
- **Complete auditability** maintains full transparency through AWS CloudTrail logging for all rotation activities, secret value updates, and management operations.