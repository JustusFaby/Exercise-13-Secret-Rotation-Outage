# Exercise 13 – Secret Rotation Outage

## Objective

Investigate and identify why secret rotation from AWS Secrets Manager did not propagate to Kubernetes, resulting in application authentication failures.

---

## Incident Scenario

Following a secret rotation in AWS Secrets Manager, the payment application started returning:

```text
401 Unauthorized
Token validation failed
```

The Kubernetes Secret used by the application had not been updated and still contained the old authentication token.

---

## Architecture

```text
AWS Secrets Manager
        │
        ▼
External Secrets Operator
        │
        ▼
Kubernetes Secret (payment-secret)
        │
        ▼
Payment Application
```

---

## Problem Statement

The application relies on a token stored in AWS Secrets Manager. After the token was rotated, the updated value was expected to be synchronized automatically to Kubernetes through the External Secrets Operator.

However, synchronization failed, causing the application to continue using an outdated token and resulting in authentication errors.

---

## Investigation Steps

### Verify External Secret Status

```bash
kubectl describe externalsecret payment-secret
```

Observed Error:

```text
could not get secret data from provider
ClusterSecretStore "aws-secretsmanager" is not ready
```

### Verify ClusterSecretStore Status

```bash
kubectl describe clustersecretstore aws-secretsmanager
```

Observed Error:

```text
failed to refresh cached credentials
no EC2 IMDS role found
```

### Verify External Secrets Operator Logs

```bash
kubectl logs -n external-secrets deployment/external-secrets
```

Observed Error:

```text
ClusterSecretStore "aws-secretsmanager" is not ready
```

---

## Root Cause Analysis

The External Secrets Operator was unable to authenticate with AWS Secrets Manager because valid AWS credentials were not available.

As a result:

1. ClusterSecretStore became unavailable.
2. External Secrets Operator could not retrieve the updated secret.
3. Kubernetes Secret was not refreshed.
4. Application continued using the old token.
5. Authentication requests failed with HTTP 401 Unauthorized.

---

## Impact

* Secret synchronization stopped.
* Kubernetes Secret became stale.
* Application authentication failed.
* Service experienced outage due to invalid credentials.

---

## Resolution

1. Configure valid AWS authentication using IAM Roles, IRSA, or AWS credentials.
2. Verify ClusterSecretStore status is Ready.
3. Confirm ExternalSecret synchronization succeeds.
4. Verify Kubernetes Secret is updated.
5. Restart application pods to load the latest secret.

Example:

```bash
kubectl rollout restart deployment payment-service
```

---

## Commands Used

```bash
kubectl describe externalsecret payment-secret

kubectl describe clustersecretstore aws-secretsmanager

kubectl logs -n external-secrets deployment/external-secrets
```

---

## Expected Outcome

After proper AWS authentication is configured:

* External Secrets Operator successfully retrieves secrets.
* Kubernetes Secret is updated automatically.
* Application uses the latest token.
* Authentication requests succeed.

---

# Demo Video

**Demo Link:**
https://drive.google.com/file/d/1rfrIGVmJTmaZGSkrWpMwFXD-EV6kWzG8/view?usp=sharing

---

## Conclusion

The outage occurred because secret rotation could not propagate from AWS Secrets Manager to Kubernetes. Investigation revealed that the External Secrets Operator lacked valid AWS credentials, preventing synchronization and causing the application to use an outdated token. Once authentication is corrected and secrets are synchronized, the application functions normally.
