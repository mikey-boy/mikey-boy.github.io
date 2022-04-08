---
layout: post
title:  "Implementing RBAC policies in Kubernetes"
date:   2022-04-08 10:29:27 -0400
categories: kubernetes rbac helm multi-tenancy
---

## Overview

In this blog post I will talk about implementing RBAC policies within a Kubernetes cluster to enforce multi-tenancy isolation. I essentially wanted to prevent different development teams working within the same cluster from stepping on eachother's toes (which can happen quite easily if everyone interacts with the cluster using cluster-admin priveleges). I achieved this by creating a custom Helm chart that creates and tracks all the necessary Kubernetes objects needed to enforce this isolation.

## Background

### Kubernetes RBAC objects

RBAC policies consist of a few basic objects:
- Roles
- RoleBindings
- ClusterRoles
- ClusterRoleBindings

If you are unfamiliar with these objects (or any other Kubernetes objects for that matter), I suggest you read [their official documentation](https://kubernetes.io/docs/reference/access-authn-authz/rbac/). One thing to keep in mind is that the objects are _additive_ in nature, thus if no RBAC objects exist everything is permitted.

### Authentication

_Authorization_ is the process of verifying whether a user is allowed to do a particular action. This is done in the cluster using the objects mentioned above. _Authentication_ is the process of verifying a user's identity, and is typically done with a login page. Kubernetes does not provide authentication natively, but instead relys upon an external authentication provider, like Azure Active Directory. The entire flow, at a very high-level, is something like this:

1. You login/authenticate with an external provider. With Azure this is done using `az login`
2. The external provider returns a token to you which identifies _your_ user
3. Whenever you interact with the cluster, with `kubectl` for example, you submit this token
4. The Kubernetes API verifies if the token is legit, and if it is, figures out who you are
5. It then checks any RBAC objects associated with _your_ user to determine whether that action should be allowed/authorized

Setting up authentication is typically done when the cluster is created. For this blog, all you need to keep in mind is that there will be small nuances in the role bindings depending on which authentication provider you are using.

### Multi-tenancy isolation

As stated earlier, multi-tenancy isolation prevents different development teams working within the same cluster from stepping on eachother's toes. One way this can be achieved is with the use of _namespaces_. For each tenant/dev-team, we can create a list of associated namespaces and grant them permissions only within those  namespaces. This will prevent others from accidently (or maliciously) affecting the objects within their namespaces and vice-versa.

Some applications can not be strictly bound to namespaces though. An nginx ingress controller needs to be able to monitor Ingress objects in _every_ namespace in order to properly route to the applications running inside the cluster.

With these requirements in mind, in order to appropriately implement multi-tenancy isolation we need the following abilities:
1. Lock down particular namespaces to a tenant
2. Allow multiple tenants to access the same namespace (if required)
3. Allow some applications to be granted limited cluster-wide permissions

## Forseen issues

Now that we have went over some base concepts and defined our requirements for success, lets look at some of the problems we might encounter. In my view, the problem is two-fold:
- Maintainability - Being able to quickly understand what permissions are granted and adjust them as necessary
- Completeness - Implementing all the different policies which will both enable developers to do their jobs _and_ ensure security within the cluster

These two issues are closely related. If we have better maintainability, we have a better view into what policies are being enforced, and we can adjust them to ensure that we are not being overly permissive/restrictive.

## Solution

### Custom Helm Chart

A custom helm chart helps with the problem of maintainability. We simplify the creation of numerous RBAC objects, many of them similar in nature, into a single values file. We update this values file as needed, and use it to reference what permissions are currently granted within our cluster.

Lets take a look at an example:
{% highlight yaml %}
tenant: team1

namespaces:
  install:  # Namespaces to be created
    - ns-1
    - ns-2
  exists:   # Namespaces that are associated with this tenant but already exist
    - ns-3

groups:
  - name: team1-sg
    id: 48102a86-af8a-23e5-9de7-ccd42a46ef03  # Azure group ID
    bindings:
      - role: view        # Using ClusterRole 'view'
        scope: cluster    # create a ClusterRoleBinding

      - role: admin       # Using ClusterRole 'admin'
        scope: namespace  # create a RoleBinding
        namespaces:       # in the associated namespaces
        - ns-1
        - ns-2
        - ns-3

  - name: team2-sg
    id: 39857f96-ac9a-21d6-8fb5-adf23b56aa21  # Azure group ID
    bindings:
      - role: view        # Using ClusterRole 'view'
        scope: namespace  # create a RoleBinding
        namespaces:       # in the associated namespaces
        - ns-1

{% endhighlight %}

From this values file, 7 different Kubernetes objects are created:
- 2 Namespaces (ns-1, ns-2)
- 1 ClusterRoleBindings (view)
- 4 RoleBindings (admin on ns-1, ns-2, ns-3. view on ns-1 for team2-sg)

Note the _id_ fields of each security group. This ID corresponds to an Azure Active Directory group, and is therefore specific to using Azure AD as the Kubernetes authentication provider.

### Custom Roles

### Directory structure
