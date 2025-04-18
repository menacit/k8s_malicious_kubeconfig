<!--
SPDX-FileCopyrightText: © 2025 Menacit AB <foss@menacit.se>
SPDX-License-Identifier: CC-BY-SA-4.0
-->

# k8s\_malicious\_kubeconfig
_TL;DR: PoC demonstrating code execution through a crafted kubeconfig__


## Introduction
Let's say you wanna let those pesky developers access Kubernetes - what are your options?
Should we perhaps generate client certificates and embed them together with their sensitive private
keys in kubeconfig files that we somehow share with our users? A possibility for sure, but not the
best choice due to the distribution hurdles, being single-factor authentication and
[lacking support for revocation checks](https://github.com/kubernetes/kubernetes/issues/18982).
  
Furthermore, IAM is nasty business that we rather let someone else deal with. Most organizations
already have a centralized solution in place. Something based on OpenID Connect, LDAP or similar.  
  
To support usage of these, Kubernetes can validate JWTs and for anything else we can think of,
there is always ["Webhook token authentication"](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#webhook-token-authentication).

With this in mind, we know that the server-side can integrate with third-party identity providers.
But how does a user who wanna run `kubectl get pods` provide their authentication information?
Tools like kubectl don't know how to ask a user to insert some exotic multi-factor key fob.

To help solve this challenge, the official "client-go" library (used by applications like kubectl)
and its Python sibling has supported "exec-based authentication plugins" since version \~1.22.
This feature allows a kubeconfig file to define an arbitrary command that (once executed) is
expected to dynamically return an authentication token on stdout that will be sent to Kubernetes. 
The executable could contain custom code to prompt for login details, relay these to an identity
provider, etc. For an example consumer of the feature, have a look at the excellent
["kubelogin" plugin](https://github.com/int128/kubelogin).  
  
While useful, this functionality could also be abused by malicious actors. An attacker who has
crafted a kubeconfig file could potentially trick victims into using it and thereby execute
arbitrary code on their device. This may seem far-fetched, but many developers are used to download
these in order to access new clusters - how often do you think they audit them first?
Besides, it's not only humans who utilize kubeconfig files - some CI/CD tools do it as well.

This repository contains a kubeconfig called "demo.yaml" - a silly proof of concept demonstrating 
how to abuse this feature. It could trivially be modified to exfiltrate secrets, download a
dynamic payload over HTTP and similar!


## Acknowledgements
This proof of concept was created during research for
[Menacit's Kubernetes Security Course](https://github.com/menacit/kubernetes_security_course).
Funding for development of the course was provided by _Sweden's National Coordination Centre for
Research and Innovation in Cybersecurity_, _the Swedish Civil Contingencies Agency_ and
_the European Union's European Cybersecurity Competence Centre_.  


## Example usage
Execute kubectl with specified kubeconfig and observe the output:

```
$ kubectl --kubeconfig demo.yaml cluster-info

Code is being executed through loading of kubeconfig
[...]

$ cat /tmp/code_exec_proof

Told you so :-)
```
