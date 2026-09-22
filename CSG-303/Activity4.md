# Activity 4: Kubernetes RBAC and Least-Privilege Access

## Overview

In this activity, you will protect a Kubernetes application by creating different access levels for three identities. Each identity will receive only the permissions required for its assigned role.

You will use Kubernetes ServiceAccounts as lab identities, Roles or ClusterRoles to describe permitted actions, and RoleBindings to connect identities to permissions. You will then test both successful and denied operations and prove that access is restricted to the intended namespace.

This activity runs entirely in the free [Killercoda Kubernetes playground](https://killercoda.com/playgrounds/scenario/kubernetes). It provides requirements, expected access decisions, and documentation links but intentionally provides **no commands, command syntax, Kubernetes manifests, or completed RBAC rules**. You must design the policies from the requirements.

## Learning objectives

By the end of this activity, you should be able to:

- distinguish authentication from authorization;
- explain the purpose of Kubernetes ServiceAccounts;
- identify the subjects, resources, actions, and scope involved in an RBAC decision;
- create namespace-scoped permissions with Roles and RoleBindings;
- reuse a ClusterRole within a single namespace through a RoleBinding;
- apply the principle of least privilege;
- distinguish between access to a resource and access to one of its subresources;
- verify allowed and denied operations for another identity;
- diagnose an incorrect RBAC binding; and
- explain how Kubernetes RBAC differs from Google Cloud IAM.

## Prerequisites

Before starting, you should:

- understand Pods, Deployments, Services, ConfigMaps, Secrets, and namespaces;
- understand that Kubernetes API actions include reading, creating, updating, and deleting resources;
- have a free [Killercoda account](https://killercoda.com/); and
- be able to inspect Kubernetes resource status and application logs.

No Google Cloud account, paid service, or new application image is required.

## Scenario

A team runs an application in a shared Kubernetes cluster. Three identities require different access levels:

### Application viewer

The viewer needs to inspect the application's normal resources but must not change anything or read confidential data.

### Application developer

The developer needs to inspect the application, view its logs, update its Deployment, and remove a failed application Pod so that Kubernetes can replace it. The developer must not read Secrets or manage access policies.

### Namespace administrator

The namespace administrator needs broad control over application and access-control resources, including Secrets, but only inside the application's namespace. This identity must not receive cluster-wide administrative access.

Your task is to implement these access levels and prove that Kubernetes permits and denies the correct operations.

## Required environment

Create the following logical structure:

- one namespace containing the protected application;
- one second namespace used to test the access boundary;
- one simple Deployment in each namespace;
- one Service, ConfigMap, and fictional Secret in the protected namespace;
- three ServiceAccounts in the protected namespace; and
- the Roles, ClusterRoles, and RoleBindings required to implement the access model.

Use only fictional data in the Secret. Do not place a real password, API key, access token, or personal credential in Killercoda.

## Required authorization outcomes

Your final RBAC design must produce the following results in the protected application namespace:

| Operation | Viewer | Developer | Namespace administrator |
|---|---:|---:|---:|
| View Pods | Allowed | Allowed | Allowed |
| View Deployments | Allowed | Allowed | Allowed |
| View Services | Allowed | Allowed | Allowed |
| View ConfigMaps | Allowed | Allowed | Allowed |
| View application logs | Denied | Allowed | Allowed |
| Update the application Deployment | Denied | Allowed | Allowed |
| Delete an application Pod | Denied | Allowed | Allowed |
| Create a new Deployment | Denied | Denied | Allowed |
| Read Secrets | Denied | Denied | Allowed |
| Modify Roles or RoleBindings | Denied | Denied | Allowed |

All three identities must be denied access to application resources in the boundary-test namespace unless a separate binding explicitly grants that access.

The table defines the required result, not the policy implementation. You must determine the necessary API resources, actions, subresources, and bindings from the Kubernetes documentation.

## Activity workflow

### Part 1 — Understand the authorization model

Before creating policies, study how Kubernetes makes an access decision.

Be able to explain:

- how an identity becomes authenticated;
- how authorization determines whether an authenticated identity may perform an action;
- what a ServiceAccount represents;
- the difference between a Role and a ClusterRole;
- the difference between a RoleBinding and a ClusterRoleBinding;
- why a ClusterRole referenced by a RoleBinding can still be limited to one namespace;
- why RBAC permissions are additive; and
- why Kubernetes RBAC does not provide explicit deny rules.

Use these references:

- [Kubernetes authentication](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)
- [Kubernetes authorization](https://kubernetes.io/docs/reference/access-authn-authz/authorization/)
- [Using RBAC authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Kubernetes ServiceAccounts](https://kubernetes.io/docs/concepts/security/service-accounts/)

### Part 2 — Prepare isolated namespaces

1. Open the [Killercoda Kubernetes playground](https://killercoda.com/playgrounds/scenario/kubernetes).
2. Wait for the cluster to become ready.
3. Create one namespace for the protected application.
4. Create a second namespace for boundary testing.
5. Give both namespaces names that clearly communicate their purpose.
6. Confirm that the namespaces are separate scopes for namespaced resources.

Use this reference:

- [Kubernetes namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)

Do not perform this activity in the default or system namespaces. Do not modify existing cluster components or their permissions.

### Part 3 — Create the test resources

1. Create a small Deployment in the protected namespace.
2. Create an associated Service.
3. Create a ConfigMap containing ordinary test configuration.
4. Create a Secret containing only fictional data.
5. Create a different small Deployment in the boundary-test namespace.
6. Verify that both Deployments are healthy before adding RBAC policies.

The application itself is not the focus of this activity. You may use an appropriate public image without creating new application code.

### Part 4 — Create the three identities

1. Create a separate ServiceAccount for the viewer.
2. Create a separate ServiceAccount for the developer.
3. Create a separate ServiceAccount for the namespace administrator.
4. Keep all three ServiceAccounts in the protected application namespace.
5. Record the full Kubernetes identity represented by each ServiceAccount.

Do not use the namespace's default ServiceAccount as one of the three test identities. Dedicated ServiceAccounts make ownership and permissions easier to understand and audit.

### Part 5 — Design the viewer permissions

1. Translate the viewer requirements from the authorization table into Kubernetes resources and permitted actions.
2. Create namespace-scoped permissions that allow read-only inspection of the required ordinary resources.
3. Bind those permissions only to the viewer ServiceAccount.
4. Do not grant access to Secrets, logs, workload modification, or RBAC resources.
5. Avoid wildcard permissions.

Pay attention to the difference between viewing Pod objects and viewing their log subresource. Access to one does not automatically mean access to the other.

### Part 6 — Design the developer permissions

1. Translate the developer requirements into the minimum required resources, subresources, and actions.
2. Permit the required read access.
3. Permit application log access.
4. Permit the required Deployment changes without granting unrestricted workload administration.
5. Permit deletion of application Pods so that their controller can replace them.
6. Keep Secrets and RBAC resources unavailable.
7. Bind the permissions only to the developer ServiceAccount.

Consider whether every action commonly associated with Deployments is actually required. Grant only what the activity demands.

### Part 7 — Design the namespace-administrator permissions

1. Research the standard user-facing roles included with Kubernetes.
2. Select an appropriate reusable role for namespace administration rather than granting cluster-admin access.
3. Bind it to the namespace-administrator ServiceAccount using a namespace-scoped binding.
4. Confirm that the binding grants broad access inside the protected namespace but not across the cluster.
5. Verify whether the selected standard role satisfies every required result in the authorization table.

Use these references:

- [Default RBAC roles and role bindings](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#default-roles-and-role-bindings)
- [RBAC good practices](https://kubernetes.io/docs/concepts/security/rbac-good-practices/)

### Part 8 — Verify permissions before performing operations

Use Kubernetes authorization checking and identity impersonation to evaluate permissions as each ServiceAccount.

1. Test every viewer decision in the authorization table.
2. Test every developer decision.
3. Test every namespace-administrator decision.
4. Repeat relevant read tests against the boundary-test namespace.
5. Confirm that the answers match the required outcomes before conducting real operations.

Use these references:

- [Check API access](https://kubernetes.io/docs/reference/access-authn-authz/authorization/#checking-api-access)
- [Authorization permission checking](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_auth/kubectl_auth_can-i/)
- [User impersonation](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#user-impersonation)

The Killercoda session initially provides a highly privileged administrative identity. Use that identity only to build the lab and perform authorized impersonation checks. Its own permissions are not evidence that your ServiceAccounts have been configured correctly.

### Part 9 — Verify selected permissions with real operations

Authorization checks predict decisions, but selected permissions must also be tested through controlled actions.

1. As the viewer identity, inspect allowed resources and attempt one prohibited modification.
2. As the developer identity, inspect application logs and perform an allowed Deployment change.
3. As the developer identity, remove one application Pod and verify that the Deployment replaces it.
4. As the developer identity, attempt to read the fictional Secret and confirm that access is denied.
5. As the namespace administrator, perform one operation that is denied to the other identities.
6. Attempt a protected-namespace operation against the boundary-test namespace as each identity and confirm that it is denied.
7. Restore the application to its original healthy state after testing.

Use only reversible actions on the resources created for this activity.

### Part 10 — Diagnose an incorrect binding

1. Create a temporary test identity with no permissions.
2. Create a deliberately incorrect RoleBinding intended to grant it viewer access. Introduce only one error.
3. Verify that the expected permission is missing.
4. Inspect the binding's namespace, subject, and role reference to locate the error.
5. Correct the binding.
6. Verify that the intended viewer permission now works.
7. Remove the temporary identity and binding after the test.

Possible categories of error include an incorrect subject, namespace, or role reference. Select and diagnose one category yourself.

### Part 11 — Review the effective access model

Review all permissions and bindings and answer these questions:

1. Which object describes allowed actions?
2. Which object connects those actions to an identity?
3. Why can the developer view logs while the viewer cannot?
4. Why can none of the identities access the boundary-test namespace?
5. What security risk would result from replacing the namespace-scoped administrator binding with a cluster-wide binding?
6. Why should Secrets normally receive stricter permissions than ConfigMaps?
7. What would happen if an additional binding granted more access to the viewer?
8. Why is there no explicit deny rule in the policies you created?

## Kubernetes RBAC and Google Cloud IAM

This activity covers Kubernetes-native authorization. It does not reproduce Google Cloud IAM.

- A Kubernetes **ServiceAccount** is a Kubernetes identity generally used by Pods and automation.
- A Google Cloud **service account** is a Google Cloud IAM principal.
- Kubernetes RBAC controls access to Kubernetes API resources.
- Google Cloud IAM controls access to Google Cloud resources and services.
- In GKE, both systems may participate in a complete access-control design.

The Killercoda environment is sufficient for practising Kubernetes RBAC but not for testing GKE IAM integration.

## Expected outcome

At completion:

- three dedicated ServiceAccounts represent the viewer, developer, and namespace administrator;
- Roles or standard ClusterRoles define the required permissions;
- namespace-scoped RoleBindings connect each identity to the correct access level;
- the viewer can inspect ordinary resources but cannot modify them, view logs, or read Secrets;
- the developer can perform the required application operations but cannot read Secrets or manage RBAC;
- the namespace administrator has broad control only inside the protected namespace;
- all three identities are denied access to application resources in the boundary-test namespace;
- both allowed and denied decisions have been verified; and
- an intentionally incorrect RoleBinding has been diagnosed and corrected.

## Completion checklist

- [ ] I created separate protected and boundary-test namespaces.
- [ ] I created the required test resources with fictional data only.
- [ ] I created dedicated viewer, developer, and namespace-administrator ServiceAccounts.
- [ ] The viewer has only the required read permissions.
- [ ] The viewer cannot access application logs or Secrets.
- [ ] The developer can view logs and perform the required application changes.
- [ ] The developer cannot read Secrets or manage RBAC resources.
- [ ] The namespace administrator has broad access inside the protected namespace.
- [ ] The namespace administrator does not have cluster-wide administrator access.
- [ ] I avoided unnecessary wildcard permissions.
- [ ] I verified every row of the authorization table.
- [ ] I tested selected permissions with controlled real operations.
- [ ] All three identities were denied access to the boundary-test namespace.
- [ ] I diagnosed and corrected an intentionally incorrect RoleBinding.
- [ ] I restored the test application to a healthy state.
- [ ] I can explain the difference between Kubernetes RBAC and Google Cloud IAM.

---

**Completion standard:** The activity is complete when all authorization results match the required matrix, selected allowed and denied operations have been tested, access remains limited to the protected namespace, and the incorrect binding has been successfully diagnosed and corrected.
