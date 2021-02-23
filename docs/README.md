This tutorial is what I wish I had when I first started working with Kubernetes and Argo. As such, it's meant for complete beginners, so feel free to skip ahead if you know the basics.

## What will we accomplish? 
You will: 
- Learn what Argo CD and Kubernetes are and why they're useful
- Have installed Argo CD into a Kubernetes cluster
- Be able to access that Argo CD instance from a public URL and sign into it with your own custom sign on service
- Learn how to create containers from code and publish them to GitHub Container Registry
- Use Argo CD to manage itself
- Deploy your own application using Argo CD

## Assumptions
I assume you:
- Are using a Mac. However, this tutorial can easily be adapted to Linux
- Have used a command line before
- Have done at least a little coding before
- Have a GitHub account
- Have basic Git skills
- Know what a Virtual Machine is
- Know what an API and a web application are
- Have basic knowledge of the Internet and networking

## Contents

0. K8S Basics
1. Basic Setup and Installing Argo CD
2. Setting up SSO
3. Publishing containers to use in your application
4. Configuring application YAML
5. Deploying application with Argo CD