## Kubernetes Cluster Management at Scale with Gardener

### 1. The Scenario

![Platform Bottleneck](platform-bottleneck.png)

Let's assume we have a large software company, more than 5,000+ employees, multiple products, customers all over the world.

This company has 50+ development teams. Each team owns different projects, different microservices, and has different needs:

- Team A builds customer-facing APIs that handle millions of requests per day

- Team B works on internal data pipelines that process terabytes of information

- Team C maintains the company's e-commerce platform

- Team D is building a new AI/ML service

And so on, and so on...

Every single team needs one thing in common: **a place to run their containers.**

#### 1.1. The Daily Conversation


Now, here's a conversation that happens every single day, multiple times a day, in this company:


**Developer:** "Hey Platform Team, we're spinning up a new service and we need a Kubernetes cluster."

**Platform Team:** "Sure, what do you need?"

**Developer:** "Nothing fancy:

- 3 worker nodes

- 8GB RAM and 4 vCPU per node

- Needs to auto-scale

- We don't care if it's AWS, Azure, or GCP, just give us something that works

- Oh, and when can we have it?"

**Platform Team:** (sighs internally) "We'll get back to you."


This exact conversation plays out multiple times a day, every single day of the year. And it's not just about creating clusters, it's about keeping them healthy, updating them, scaling them, and making sure they don't break at 2 AM.

### 2. The Problem

![Problem](problem.png)

Every team just wants one thing: a place to run their containers. So they ask the platform team for a Kubernetes cluster. Simple, but here's where it gets hard.

1. **Provisioning is the easy part**: Terraform or eksctl can spin up a cluster in minutes. That's day one. The real work is day two: upgrades, patches, backups, cert rotation, and 2 AM fixes, all of it stays with the platform team forever.

2. **Consistency is hard**: Build 50 clusters over time and they slowly drift — different versions, different configs. Keeping a whole fleet identical and healthy is a constant fight.
Every cloud is different. AWS, Azure, and GCP each have their own tools and rules. The platform team has to know them all.
3. **It doesn't scale**: Five clusters is fine. Fifty across three clouds is not. The platform team becomes the bottleneck.

In the end, teams wait, the platform team drowns, and clusters quietly turn into a risk instead of a tool.

### 3. What We Actually Need?

If we want to fix the problem mentioed above, we need a way of managing clusters that does four things:

![Needs](needs.png)