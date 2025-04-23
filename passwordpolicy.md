
# How to Secure Your CPK PostgreSQL Database with the CrunchyData Password Policy Extension

**CrunchyData Postgres for Kubernetes (CPK)** is a comprehensive solution designed to simplify deploying, managing, and scaling PostgreSQL on Kubernetes clusters. Integrating **passwordpolicy**, an extension provided by CrunchyData, can further enhance your database's security by enforcing complex password requirements. This guide will walk you through setting up and configuring this powerful extension.

## Prerequisites

Before you begin, ensure that:

1. You are using a **currently supported version** of CPK.
2. Your PostgreSQL cluster is running inside a Kubernetes environment.
3. Familiarity with Kubernetes resources such as Deployments and ConfigMaps is beneficial.

## Step 1: Verify the Password Policy Extension Availability

Ensure that your currently supported version of CPK includes the **passwordpolicy** extension. You can check the availability using the following command within an interactive PostgreSQL shell:

```sql
SELECT name, comment FROM pg_available_extensions WHERE name = 'passwordpolicy';

      name      |                     comment
----------------+--------------------------------------------------
 passwordpolicy | passwordpolicy - strengthen user password checks
(1 row)
```

If you do not see `passwordpolicy` listed, it may be necessary to update your PostgreSQL image or version.

## Step 2: Enable the Password Policy Extension

The first step involves enabling the passwordpolicy extension by modifying your `PostgresCluster` manifest.

### Step-by-Step Guide

#### Step 2.A: Modify the Manifest to Use the Appropriate Image

Ensure that you are using a PostgreSQL image that includes the **passwordpolicy** extension. Typically, newer releases of CPK include this feature.

```yaml
apiVersion: postgres-operator.crunchydata.com/v1beta1
kind: PostgresCluster
metadata:
  name: secure-pg-cluster
  namespace: postgres-operator
spec:
  postgresVersion: "17"
```

#### Step 2.B: Apply the Manifest

Deploy your changes using `kubectl`:

```bash
kubectl apply -f /path/to/secure-pg-cluster.yaml
```

## Step 3: Configure Password Policies Using Dynamic Configuration

You can configure password policy parameters through dynamic configuration within the PostgreSQL cluster’s manifest. This allows you to enforce specific requirements such as minimum password length and character types.

### Example Manifest with Configured Policies

```yaml
apiVersion: postgres-operator.crunchydata.com/v1beta1
kind: PostgresCluster
metadata:
  name: secure-pg-cluster
  namespace: postgres-operator
spec:
  postgresVersion: "17"
  instances:
    - name: instance1
      dataVolumeClaimSpec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 1Gi

  port: 5432
  patroni:
    dynamicConfiguration:
      postgresql:
        settings:
          - name: shared_preload_libraries
            value: "passwordpolicy"
          - name: p_policy.min_password_len
            value: "6"
          - name: p_policy.min_special_chars
            value: "1"
          - name: p_policy.min_numbers
            value: "3"
          - name: p_policy.min_uppercase_letter
            value: "1"
          - name: p_policy.min_lowercase_letter
            value: "2"
```

**Patroni Password Policy Settings**:
   - `shared_preload_libraries`: Load the passwordpolicy extension during PostgreSQL server startup.
   - `p_policy.min_password_len`: Minimum password length, e.g., 6 characters.
   - `p_policy.min_special_chars`: Minimum number of special characters required.
   - `p_policy.min_numbers`: Minimum number of numeric characters required.
   - `p_policy.min_uppercase_letter`: Minimum number of uppercase letters required.
   - `p_policy.min_lowercase_letter`: Minimum number of lowercase letters required.

#### Step 3.A: Apply the Updated Manifest

```bash
kubectl apply -f /path/to/secure-pg-cluster.yaml
```

#### Step 3.B: Verify Settings Applying Correctly

Check to confirm that these settings apply correctly:

```sql
SELECT name, setting, context FROM pg_settings WHERE name LIKE 'p_policy.%';
             name              | setting | context 
-------------------------------+---------+---------
 p_policy.min_lowercase_letter | 2       | sighup  
 p_policy.min_numbers          | 3       | sighup  
 p_policy.min_password_len     | 6       | sighup  
 p_policy.min_special_chars    | 1       | sighup  
 p_policy.min_uppercase_letter | 1       | sighup  
(5 rows)
```

## Step 4: Testing Password Policies

To ensure the password policies are functioning as expected, attempt creating new users with different password strengths.

### Example Test Cases and Expected Errors

```sql
-- Attempt to create user with a short password
postgres=# CREATE USER pb WITH PASSWORD 'W';
ERROR:  password is too short.

/* Add more complexity with lowercase */
postgres=# CREATE USER pb WITH PASSWORD 'welc0me!';
ERROR:  password must contain at least 1 uppercase character.

/* Ensure it includes numbers and special characters */
postgres=# CREATE USER pb WITH PASSWORD 'Welcome!';
ERROR:  password must contain at least 3 numeric characters.

/* A stronger password that meets all criteria */
postgres=# CREATE USER pb WITH PASSWORD 'W3lc0m3!_HdshD*hsf44w&*(FH9';
CREATE ROLE
```

Feel free to adjust the manifest settings based on your specific security requirements after observing which configurations enforce appropriate complexity and strength for passwords.


### Step 6: Automated Password Policy Checks with Scheduled Jobs

Set up scheduled cron jobs to verify password compliance periodically, alerting administrators or triggering automated password resets if non-compliant passwords are detected. This can be implemented using an external script running within your environment.

## Conclusion

Using the **passwordpolicy** extension in CPK greatly enhances the robustness of your PostgreSQL database security by enforcing stringent password requirements. By following the steps outlined in this guide, you can effectively manage and enforce strong password practices across your Kubernetes-managed databases.

For more detailed information and advanced configurations, refer to the [CrunchyData Postgres for Kubernetes documentation](https://access.crunchydata.com/documentation/postgres-operator/latest/). If needed, consult the specific page on the [passwordpolicy extension here](<https://access.crunchydata.com/documentation/passwordpolicy/latest/>).

