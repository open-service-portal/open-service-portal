# Cloud-Native Service Marketplace on Kubernetes: Concept and Architecture

## 📋 Table of Contents

1. [Introduction and Goals](#introduction-and-goals)
2. [Marketplace Architecture](#marketplace-architecture)
3. [Self-Service Interfaces](#self-service-interfaces)
4. [Security, Isolation, and Governance](#security-isolation-and-governance)
5. [Risks and Challenges](#risks-and-challenges)
6. [Alternatives and Comparison](#alternatives-and-comparison)
7. [Conclusion](#conclusion)

## 🎯 Introduction and Goals

An internal cloud marketplace should let developers (internal customers) order complex services via self-service, without having to deal with the technical details of the infrastructure. Everything should be implemented cloud-native — at its core based on Kubernetes. Even external resources like DNS records or firewall rules are modeled and managed as Custom Resources (CRDs) in Kubernetes. The vision is essentially an Internal Developer Platform (IDP): developers order a service via API, CLI, or web UI, and in the background all required components and infrastructure are provisioned automatically. This increases developer autonomy and relieves central teams, since no manual provisioning processes are needed anymore.

## 🏗️ Marketplace Architecture

### Technology Stack
- **Kubernetes** + **Crossplane** + **GitOps**

### Kubernetes as the Control Plane
The marketplace is based on a Kubernetes cluster that serves as the central control plane for all services. Each service is represented by a Kubernetes Custom Resource (CR). Thanks to Crossplane — an open-source project for extending Kubernetes toward infrastructure — we can define and manage cloud resources and even non-Kubernetes components as CRDs. To do so, Crossplane installs so-called provider controllers for various cloud and infrastructure APIs and lets us control these external resources declaratively through Kubernetes.

### Everything as a CRD

For every offered service (whether a "complex" service like a data science platform or a "component" service like a PostgreSQL database) a dedicated CRD type is defined. Even infrastructure building blocks like DNS records, certificates, or firewall rules are mapped into Kubernetes via Crossplane providers, so that all building blocks in the marketplace can be managed through the Kubernetes API. This API-centric model avoids media breaks: developers interact exclusively with Kubernetes objects, not with separate cloud consoles or scripts. Credentials for cloud providers live only with Crossplane itself (e.g. as a ProviderConfig with an attached Secret), so developers need no cloud credentials and permissions are managed centrally in Kubernetes.

### Crossplane Compositions – Abstraction for Service Owners

A central concept is abstraction through Crossplane Compositions. Each service type gets a user-friendly surface API (Custom Resource Definition) whose spec contains only the essential input parameters. The implementation details are captured in a Composition that defines which concrete resources are created in the background. This allows complicated setups to be abstracted into higher-level resources. For example, there might be a CRD type `PostgreSQLInstance` where the user only specifies parameters like database size and version — things like VM instance size, networking, or backup settings are hard-coded in the Composition by the service owner (platform team) and not visible to the user. Crossplane then takes care of creating all required cloud resources when such a CR is created (e.g. an AWS RDS instance or an Azure PostgreSQL DB), possibly including supplementary resources like subnets, DNS records, or IAM users. To the developer this looks like a single resource in Kubernetes that they create — the complex details are hidden from them by the Composition. This principle can also be nested: a Composition can create multiple underlying resources — for example, provisioning an AI data science platform could simultaneously create a PostgreSQL database, S3 storage, and a Kafka cluster, by having the Composition internally create the corresponding component services as CRs.

### Service Owners and Layering of Services

This results in a multi-tier model of services, each with its own responsible party (service owner):

#### 1️⃣ Infrastructure Services

Examples: "Kubernetes cluster", "DNS record", "TLS certificate", "firewall rule". These are mostly provided by the central platform team. They form the lowest layer and often run as Crossplane managed resources directly on cloud APIs (e.g. DNS via Route53, clusters via EKS/AKS, etc.).

#### 2️⃣ Component Services

Reusable technical services like PostgreSQL, MongoDB, Kafka, S3 storage, etc. These have their own service owners (database team, messaging team, …) who are experts for their service. They use the infrastructure services as building blocks: for example, the PostgreSQL service might need a Kubernetes cluster or a VM to run the DB on, persistent storage, and possibly a DNS record for the DB URL. The DB service owner wires all of this in via Crossplane without having to operate these infrastructure details manually — they only declare in their Composition that, say, a `DNSRecord` CR should be created for the DB, and rely on the DNS service owner having ensured that `DNSRecord` CRDs work. This way every component service can model its dependencies as further CRs, which in turn are delivered by other owners.

#### 3️⃣ Complex Services

These are the offerings that the end user (developer) ultimately orders, e.g. an AI data science platform or, more generally, a development environment, a Jenkins-as-a-Service, etc. Internally they consist of multiple component services but are offered as a single product. The service owner of a complex service defines its CRD and Composition such that creating, e.g., an `AIPlatform` resource automatically draws on all required components (Postgres, Kafka, … see above). An important acceptance criterion here: the complex-service owner should ideally not need to know anything about the implementation of the component services. In the Composition they only state, e.g., "create a PostgreSQL instance for this AIPlatform" (by creating a `PostgreSQLInstance` CR with certain parameters). Exactly how the DB service provisions that instance stays encapsulated in its Composition. Likewise, the component-service owner (e.g. for PostgreSQL) doesn't need to know the underlying infrastructure in detail; they request infrastructure via, e.g., a cluster CR or storage-volume CR, without configuring VMs directly — those details again sit in the infra service Compositions. So everyone only cares about their own abstraction.

Through this layering and Crossplane, complexity stays in the lower layers (platform team), while the upper layers have simple interfaces. Developers can thus order complex applications without knowing about the many dependencies behind them — "Everything as a Service" within the company. Ordering the AI platform, for instance, just requires creating one CR; from it, Crossplane creates the DB, the Kafka, all necessary infrastructure objects, and of course the AI platform service itself — fully automated.

### Deploying the Services via GitOps

The service definitions (CRDs, Compositions, etc.) live in Git repositories — e.g. each service has its own GitHub repo with its YAML definitions. A GitOps tool like Flux or Argo CD continuously syncs these into the cluster. That means when a service owner evolves their offering (new Composition version, new CRD fields, etc.), a single commit suffices and the GitOps operator applies the changes in the Kubernetes cluster. This keeps the control-plane cluster always up to date with the desired service-catalog definitions. It greatly simplifies updates and versioning, since all changes are traceably recorded in Git history.

**Example**: The owner of the PostgreSQL service maintains, in their Git repo, the Crossplane Composition that creates an AWS RDS Postgres. Flux syncs the CRD and Composition into the cluster. When they make an optimization (e.g. a different instance type or backup policy), they commit the change; Flux updates the Composition. New orders then automatically use the updated definition.

```yaml
# Example of a simplified PostgreSQL Composition
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: postgresql-aws
spec:
  compositeTypeRef:
    apiVersion: database.company.io/v1alpha1
    kind: PostgreSQLInstance
  resources:
    - name: rds-instance
      base:
        apiVersion: rds.aws.crossplane.io/v1alpha1
        kind: DBInstance
        spec:
          forProvider:
            region: eu-central-1
            engine: postgres
            engineVersion: "14"
            dbInstanceClass: db.t3.micro
            allocatedStorage: 20
```

For the prototype, any managed Kubernetes can be used (e.g. a cluster in an environment like Spot by NetApp or Rackspace — the only requirement is that Crossplane + GitOps can run on it). For production, one would then use the company cluster (or managed K8s in the cloud). The platform is in principle cloud-agnostic, as long as Crossplane providers are available for the cloud services used (AWS, Azure, GCP, and many others are supported).

## 🖥️ Self-Service Interfaces

### Overview

Developers should be able to order the services in several ways:

#### 1. Kubernetes CLI/API (kubectl)

Since everything runs through Kubernetes objects, an experienced user can simply create the desired CR in the cluster with `kubectl apply -f myservice.yaml`. This is the most direct method (or via a CI/CD pipeline). The Terraform Kubernetes provider can also be used — it lets you embed Kubernetes resources (i.e. our service CRs) into Terraform manifests and provision them that way. In both cases the user ultimately talks to the Kubernetes API.

#### 2. Web Frontend (Backstage)

For a convenient ordering interface we rely on Backstage as the developer portal. Backstage presents the catalog of available services and lets users provision a service through form inputs. To this end we integrate Backstage tightly with our Kubernetes/GitOps flow:

##### Catalog Integration

Every service type (CRD) should appear in the Backstage catalog, ideally without much manual upkeep. Backstage offers a Kubernetes Ingestor plugin for this, which automatically searches the Kubernetes cluster for certain resources and imports them as components into the catalog. In particular it can automatically ingest Crossplane claims and CRDs: i.e. our offered service CRDs appear as a template or component in Backstage without us having to hand-write a `catalog-info.yaml` for each one. Annotations on the CRDs/resources control how they are presented in the catalog. This saves a lot of manual upkeep, since the catalog is fed directly from the Kubernetes reality.

##### Ordering via GitOps

In the simplest case, Backstage could make a direct API call to the cluster to create a new CR (via a backend plugin with a ServiceAccount). A GitOps integration is better, though: the TeraSky Crossplane plugin for Backstage allows an order to create a pull request in a Git repo containing the desired custom resource. This PR can be reviewed and merged, after which Flux/ArgoCD creates the new CR in the cluster. This approach keeps all changes under version control and fits a declarative workflow (even service orders are captured as code). For the prototype, however, one can keep it pragmatic at first (KISS principle) and execute orders directly, refining the GitOps integration later.

##### Visualization

Backstage serves not only for ordering but also for an overview and the status of running services. Via the Crossplane Resources plugin we can inspect the details of an ordered service instance in the UI — including the underlying resources and even a graph visualization of the dependencies. So a developer can see in Backstage, e.g., that their AI platform X contains a PostgreSQL DB Y and a bucket Z, and whether all of these are healthy. There is also an overview card per service component with status info (running, error, etc.), which eases day-2 operations.

##### Authentication and Permissions

Backstage integrates with identity providers. For the prototype we plan to use GitHub OAuth for login (quick setup), while in production we would of course connect the company-wide Microsoft Entra ID (Azure AD). Roles and permissions can then be managed centrally (e.g. which users may order which service). For the prototype, however, simplicity comes first; complex RBAC policies are initially set aside (one can assume all internal users have equal rights in the test system).

### 🏗️ Summary of Architecture Components

The architecture consists of the following main components:

| Component | Function | Details |
|------------|----------|----------|
| **Kubernetes cluster** | Central API platform | Hosts all CRDs and the control plane |
| **Crossplane** | Infrastructure-as-Code | Defines CRDs for services, manages lifecycle |
| **GitOps** | Deployment & sync | Flux/Argo CD syncs configs and orders |
| **Backstage** | Developer portal | UI for catalog, ordering, status dashboards |
| **Identity provider** | Auth & access | GitHub (prototype) / Azure AD (production) |

Such an interplay of Backstage as the UI, Crossplane as the API layer, and GitOps as the delivery mechanism is considered a best practice in the platform-engineering community — it provides all the building blocks to deliver an internal platform that is both user-friendly (via Backstage) and operationalized (via K8s/Crossplane).

## 🔒 Security, Isolation, and Governance

With a shared Kubernetes cluster for the marketplace, it's important to handle multi-tenancy and access rights cleanly. Crossplane and Kubernetes offer mechanisms for this:

### Namespaces and RBAC

We plan to use separate namespaces per team or use case. Developers get rights only in "their" namespace, to create service instances (claims) there. The CustomResourceDefinitions of the services can be designed to support namespaced claims — i.e. the developer creates, e.g., a `DatabaseClaim` in their namespace, while Crossplane manages a cluster-wide `XDatabase` resource in the background. Via RBAC one can grant rights like "the DataScientists group may create objects of type `AIPlatform` (or its claim) in the `team-ds` namespace". But they may not, e.g., create the low-level infrastructure CRDs directly. This way only the abstracted APIs are exposed at the namespace level and the underlying details are protected.

### Isolation of Provider Credentials

Crossplane allows using different cloud credentials per namespace, by having the Composition patch the `spec.providerConfigRef` depending on the namespace. In our context this may be relevant if different projects should use different cloud accounts. In the simplest case, though, everyone uses the same account, in which case a global configuration suffices (the ProviderConfig lives in the `crossplane-system` namespace and is used by everyone). It's important that developers never see the credentials directly — only Crossplane has access, which is a security gain.

### Policy Enforcement

In a production platform you want to enforce policies, e.g. that nobody orders a DB larger than X GB or that certain tags are set. Admission controllers / policy engines like Kyverno or OPA Gatekeeper can be used for this. Crossplane itself already offers ways in Compositions to set default values or validate inputs (via the OpenAPI schema). Beyond that, cross-cutting policies could be controlled with tools like Kyverno — the Backstage plugin supports, e.g., visualizing Kyverno policy reports directly in the UI. In the prototype we'll keep policies fairly minimal (KISS), but for production operation this is an important aspect (governance).

### Separation of Responsibilities

The service-owner structure described above already brings a certain separation. Each service owner (e.g. the Postgres team) develops their Composition and has write rights to it, but another service owner does not. This can be enforced via Git repos and CI/CD (every merge into the central configs goes through code review by the respective team). Within Kubernetes, Crossplane could be run in different scoping modes — e.g. in theory multiple Crossplane instances in different namespaces for different teams (probably not needed for our use case, since a central Crossplane is fine). The key point: only trusted admins may install new CRD types (XRDs/Compositions); normal dev users may only create instances/claims. This keeps control over the "offering portfolio" with the platform team.

### Authentication and Auditing

With Azure AD as the IdP, it's ensured that only authenticated employees have access. Every action (e.g. creating/deleting a CR) is a Kubernetes API operation and therefore traceable in the audit log. In addition, GitOps maintains an audit trail of all changes (commits, PRs), which makes revisions traceable.

### Environment Separation

One may separate prototype/development from production via dedicated clusters, or at least namespaces. In the prototype cluster, one can test freely with dummy services. Later, production services could run on a dedicated Crossplane cluster that manages the real cloud resources, while developers possibly work in separate clusters. Crossplane even supports multi-cluster scenarios (control plane of control planes), for example if you want to define the service CRDs centrally but roll them out across multiple target clusters — e.g. to serve different regions or staging/prod. This complexity isn't needed for now, but it's scalable.

### Secret Management

Many services deliver credentials (DB credentials, API keys). By default Crossplane places such connection secrets in a namespace (e.g. `crossplane-system`). One has to decide how these are made available to end users. The Composition might copy the secret into the orderer's namespace (Crossplane can propagate connection details from the Composition up to parent CRs). Alternatively, Backstage could read the secrets and show them to the user, or integrate with a Vault. The important thing is to avoid security gaps here (access only for authorized users to their own secrets).

In summary, we minimize risks by applying strict RBAC rules, clearly delineated namespaces, and auditable workflows (GitOps, code reviews). This keeps the self-service portal controlled and secure, despite high autonomy for developers.

## ⚠️ Risks and Challenges

Despite the promising concept, there are several challenges and risks to consider:

### 📈 Steep Learning Curve & Cultural Change
Introducing Crossplane and the "everything as a Kubernetes object" concept requires training. Service owners must learn to write Compositions (YAML, understanding of the provider CRDs); developers must learn to work with the abstracted CRDs. While this is easier than operating Terraform/cloud APIs directly, it's still new. Without acceptance or with insufficient training, users might be tempted to bypass the platform.

### 🏗️ Abstraction Design & Maintainability
The quality of the abstracted services stands or falls with the design of the CRD interfaces. If a service owner picks the wrong parameters (too many details passed upward, or too inflexible), either usability or usefulness suffers. So guidelines are needed for service owners on which options to give their users and which to fix internally. Compositions must also be versionable — Crossplane allows, e.g., Composition Revisions, so existing instances can stay on an old version while new ones use an updated Composition.

### ⚠️ Error and Conflict Handling
With automated cascading provisioning, error management is critical. Example: a data scientist orders the AI platform, which should create, among other things, a DB and a Kafka. What happens if the DB provisioning fails (e.g. cloud quota reached)? Crossplane will record this in the status of the `AIPlatform` resource (events, conditions). The user then sees "provisioning failed". It must be clearly communicated how to proceed in such cases (retry? increase quota? involve support?). The platform should provide status info as transparently as possible (the Backstage plugin helps here, since it can show the status of each sub-resource).

### ⏳ Transient Inconsistencies
During an order, resources are created sequentially. It can take time (a DB may take several minutes). During this time the overall service is not yet ready. That's normal, but users should understand it (e.g. status "provisioning"). In addition, Crossplane must handle dependencies correctly — normally one defines implicit dependencies by waiting for provided secrets or the status of sub-resources, which Crossplane handles. Still, the Compositions must be tested carefully so that all patches & links are correct (e.g. propagating the DB URL from the RDS secret into the AIPlatform secret, etc.).

### 🚀 Performance and Scalability
A Kubernetes operator (Crossplane) has limits regarding how many resources it can manage. Crossplane is designed to manage hundreds to thousands of CRs, but the team should set up monitoring (Prometheus metrics from Crossplane) to ensure the reconciliation loops run performantly. In very large environments, splitting across multiple Crossplane instances (per environment or per team) could be considered. In the prototype this is irrelevant, but on success one must keep scalability in mind.

### 🎯 Crossplane Maturity & Provider Coverage
Crossplane itself is a CNCF incubation project (as of 2025) and is actively being developed. Many companies use it in production, but it remains a complex system. One must keep an eye on updates. The providers (e.g. for AWS, Azure) should be carefully checked for their version and stability — not every provider is equally mature. For some specialized services there may (still) be no ready-made provider. In such cases one would need workarounds (e.g. the Crossplane provider for Terraform — Provider Jet — to run a Terraform configuration if Crossplane can't do something directly). These gaps should be identified early so there are no nasty surprises if a particular service turns out not to be fully automatable.

### 🔒 Lock-in and Alternatives
Indirectly, one takes a path that depends on certain tools (Crossplane, ArgoCD, Backstage). However, these are open-source solutions run on-premises, so not a classic vendor lock-in. Still: should Crossplane not prevail in the future, or the company pursue a different strategy, one faces a migration of the platform. Fortunately Crossplane only abstracts on Kubernetes standards — i.e. in the worst case you "only" have Kubernetes manifests that you may have to interpret differently. The marketplace approach itself is implementable independently of the concrete tool.

### 💰 Resource Costs and Governance
Self-service can lead to uncontrolled resource consumption if no guardrails exist. Suddenly every developer has dozens of DB instances running. We must prepare for this: e.g. quotas per namespace, approval processes for especially expensive services, or at least transparency about running costs. Crossplane itself has no built-in cost control, but one could connect a billing export to Backstage or define alerts. This topic must be clarified organizationally (who bears costs, approvals, etc.), but it belongs to the risks.

### ♾️ Lifecycle and Cleanup
When a user no longer needs a service and deletes the CR, Crossplane ensures all subordinate resources are cleaned up (including cloud resources). That's great for automatic cleanup — but also carries risk: data could be lost if something is deleted accidentally. One may want protection mechanisms ("do not delete prod DB without approval"). Crossplane offers, e.g., a DeletionPolicy on some resources (e.g. Retain instead of Delete). This can be accounted for in Compositions. For the prototype the default is fine (delete deletes everything), but in production one must define which services have critical persistent data and possibly plan a grace period or backup before deletion.

### 🔄 CI/CD for Service Implementations
Besides the Crossplane layer, there are also the actual service components. E.g. the AI platform service might consist of a collection of microservices that exist as Docker images. These must be built (GitHub Actions) and deployed somewhere (perhaps as a Helm chart via Crossplane's Helm provider, or via ArgoCD as part of the Composition). We have dummy services, but as soon as it becomes real, we need deployment pipelines for the actual service applications. This belongs to the technical implementation — in the concept we assume service owners containerize and version their components. The marketplace then orchestrates their deployment (e.g. in a shared cluster or a dedicated cluster per instance). This is another workstream (building images, registries, etc.) that must be tackled in parallel to make the platform runnable end-to-end.

### 🎨 Backstage Integration Effort
The mentioned Backstage plugins (Kubernetes Ingestor, Crossplane UI, etc.) come from open source (e.g. from TeraSky) and must be integrated into your own Backstage. This is doable but requires some frontend/backend work in the Backstage project (installing plugins, configuring per the docs). You should plan time for it. Backstage itself must also be hosted and maintained (updates, plugin upkeep). As an alternative there is hosted Backstage (e.g. SaaS from Roadie) — there the plugins are partly already integrated. But for the prototype it's probably self-hosted Backstage in the cluster.

### 🔍 Observability and Debugging
It's been mentioned already, but again: when something goes wrong, platform teams must be able to find the cause quickly. Crossplane writes events and logs — one should have central logging/monitoring (ELK/Loki, Prometheus). There's also telemetry for performance metrics (Crossplane reconcile times, etc.). This ensures the platform team can fix problems before users give up in frustration.

Despite these challenges, the benefits outweigh them: developer satisfaction through self-service, consistent automation, and clear responsibilities per service. Many of the risks can be mitigated through policies, training, and a gradual approach (dummy services first, then critical services).

## 🔄 Alternatives and Comparison

The proposed concept relies heavily on Kubernetes and Crossplane. There are, however, alternative approaches that can be considered or that help with the evaluation:

### Kubernetes Service Catalog / Open Service Broker (OSB)
This used to be the Kubernetes way to provide external services. Via a service broker and the OSB API, developers could create ServiceInstances that then provisioned, e.g., a DB in the cloud. In OpenShift such a Service Catalog exists(ed). However, this model is now somewhat dated and the Kubernetes community has discontinued the project. Moreover, the CRDs provided by the broker were relatively generic and not well integrated into modern GitOps workflows. Our Crossplane approach achieves something similar (self-service DB, etc.) but in a Kubernetes-native way and with more flexibility in the interfaces (we can define our own CRDs instead of only the broker's predefined plans). OSB is less suitable if you have very individual internal services — it was meant more for standard cloud services.

### Direct Terraform/Pulumi Portal Solutions
Some companies build internal portals that, on click, run Terraform scripts in the background to set up infrastructure. Something like this could be implemented with, e.g., ServiceNow or a web UI + Terraform Cloud. This also fulfills the purpose of a marketplace (catalog, ordering, automation). Drawbacks: you have two worlds — Kubernetes for apps, Terraform for infra — and no shared control. Developers might still need to know the specifics of the Terraform modules, which again raises the barrier. Also missing is continuous reconciliation: Terraform runs once, whereas Crossplane as an operator constantly corrects drift and is integrated into Kubernetes. Our Kubernetes-centric approach provides a unified platform API and real-time self-service, which is harder to achieve with stand-alone Terraform (but it is certainly an alternative if you wanted to avoid Kubernetes).

### KubeVela (OAM)
KubeVela is a framework based on the Open Application Model that also puts abstract API layers over Kubernetes. It targets primarily applications/workloads but can also integrate infrastructure (e.g. via Crossplane). KubeVela lets platform teams offer predefined components and traits from which developers build their deployments. With Vela one could, e.g., define a component type "PostgresDB" that uses Crossplane internally. It also offers a UI (VelaUX). The difference: KubeVela is more focused on application rollouts and developer experience, while Crossplane focuses on infrastructure. In our case, which is about an explicit marketplace with multi-service dependencies, Crossplane fits more directly. However, KubeVela could serve as a complementary layer to unite app deployment (CI/CD) with infrastructure provisioning. For the prototype it's probably overkill, but it's good to know that OAM/KubeVela exists as an alternative in case Crossplane alone doesn't meet all wishes.

### KRO – Kubernetes Resource Orchestrator (by AWS)
Brand new (late 2024), AWS introduced an open-source project called KRO. It pursues a goal similar to Crossplane — namely defining your own platform APIs that orchestrate multiple resources — but with a different approach. Instead of writing two layers (XRD + Composition), in KRO you declare everything in a construct called a ResourceGroup that describes all components. From this, KRO automatically generates the needed CRDs and controllers at runtime. In principle it saves some complexity in the definition. However, KRO is currently experimental (beta) and not yet production-ready. Crossplane is significantly more mature. Long-term, KRO could become interesting, since it aims to manage dependencies and ordering automatically. For our decision this means: we watch KRO but stay with Crossplane for now, because stability and community support matter more than cutting-edge experiments.

### Kratix (Syntasso)
Kratix is another open-source framework that addresses exactly the "marketplace for XaaS" idea. It introduces the concept of a Promise — a promise of a service created by platform engineers. When a developer requests a Promise, Kratix ensures the necessary service is provisioned. Internally Kratix can use, e.g., Crossplane to do the implementation. Kratix's advantage: it offers marketplace mechanics out of the box and supports multi-cluster, i.e. you can have central control that then provisions services in target clusters. Syntasso (the company behind it) markets an enterprise version, but the OSS version fulfills many core functions. Kratix sees itself as "intelligent glue" between frontend and IaC backends. It lets platform teams offer everything as a service, consumable via UI, API, or CLI, while, e.g., Crossplane builds the infrastructure in the background. This is basically similar to our approach, but Kratix already provides certain structures and a community marketplace (for common Promises). As an alternative, one could consider using a framework like Kratix that simplifies this combination instead of combining everything with Crossplane + Backstage ourselves. However, one would then have to learn its concept, and the flexibility is bound to the framework. In our setup we already plan for Backstage (which integrates well with Kratix) and write our Crossplane Compositions ourselves — which is maximally flexible. Kratix may be worth evaluating if we find that many recurring patterns appear that already exist as a Kratix Promise.

### In-House Development / Operators
Finally, there's always the option of writing custom operators for each service (e.g. an "AIPlatform operator" in Go, a "Postgres operator", etc.). This would be the traditional path before Crossplane: each team implements a Kubernetes operator that manages its service. This offers full control but is laborious, since a lot of code must be written and maintained (controller logic, CRD schemas, etc.). Crossplane drastically reduces this effort by acting as a meta-operator — new APIs and their implementation are created through configuration instead of code. This spares you a flood of custom-built controllers. Given the team's resources and the desired speed, a fully custom operator approach is out — instead we use Crossplane as a framework, which has already proven efficient.

### Conclusion on Alternatives
For our use case (internal cloud marketplace, strong Kubernetes orientation), Crossplane with GitOps and Backstage is a very fitting solution, as it combines cloud-native principles (declarativity, self-service, API standardization). The alternatives mentioned show that the concept is on trend — other projects like KubeVela, Kratix, KRO aim in a similar direction, with partly different focuses. This confirms our basic idea. At the same time we should watch the market's evolution: e.g. a combination of Crossplane and KRO could become best practice in the future, or Kratix could solve some functions (multi-cluster, packaging) more conveniently. For the moment, though, we build on proven components that work well together.

## 🎯 Conclusion

The outlined concept sketches a path toward a Kubernetes-based service marketplace that offers developers a modern self-service platform. By using Crossplane as an extension of the Kubernetes API, we can hide complex infrastructure behind simple custom resources, so that each domain (databases, messaging, platform, etc.) can offer its service as an API product itself. With GitOps, consistency and traceability are guaranteed, while Backstage as a unified portal rounds out the user experience.

It's important to adapt not only the technology but also processes and culture — such as clear responsibilities (service owners), training of users, and guidelines for safe usage. We will start with a prototype (e.g. on a managed K8s cluster in a sandbox environment) and dummy services like MongoDB, PostgreSQL, Kafka, DNS, firewall, to validate these ideas at small scale. In this step we can build and evaluate the integration (CI pipelines, GitHub Actions, container deployments to Spot/Rackspace, etc.) before moving to critical production services.

All in all, this cloud-native marketplace approach promises significant benefits: faster provisioning for developers, fewer manual tickets, consistent infrastructure following best practices, and high reusability of components. The possible risks — from complexity to governance — are manageable if we account for them from the start and proceed deliberately. The concept is modularly extensible and open to new tools, so we can stay modern and flexible in the future too. This lays the foundation for an internal cloud platform that lets our developers innovate on their own, without creating chaos or uncertainty. The path is demanding, but the results will substantially improve developer experience and efficiency.
