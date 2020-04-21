# Service Control Policies

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": "secretsmanager:GetSecretValue",
      "Resource": "arn:aws:secretsmanager:*:*:secret:cve-managed-security-shared/*",
      "Effect": "Deny",
      "Sid": "DenyGetSecretValueOnCyberSecurityNamespace"
    }
  ]
}
```
