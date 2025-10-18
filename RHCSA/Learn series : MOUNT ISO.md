
### Table of contents

- [Introduction](#introduction)
- [Concept](#concept)
- [Types of mapping in autofs](#types-of-mapping-in-autofs)
- [Configuring autofs service](#configuring-autofs-service)
- [Test autofs service](#test-autofs-service)
- [Tips and outro](#tips-and-outro)

### Introduction

In this article, we walkthrough how to mount an ISO file as a repository. <br>
Normally, depending on the distros, we will register the VM with the respective subscription management service. When a VM is registered, a repository file is created automatically that will point to available online repositories, which then allow users to download and install package via the native package manager like `apt` or `dnf`.

For Red Hat Linux, we use `subscription-manager` tool to register the VM.

As root user, run `subscription-manager status` to check if VM is registered. 
<img width="9100" height="650" alt="image" src="https://github.com/user-attachments/assets/bd6c3cc2-4f85-4b29-a2b5-648ebfc3af36" />



However, there are cicumstances 

