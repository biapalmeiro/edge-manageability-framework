# Design Proposal: Add suppport for K3s

Author(s): Hyunsun Moon, Andy Bavier, Denisio Togashi

Last updated: 4/28/25

## Abstract

This proposal aims to enhance the Cluster Orchestration by introducing K3s as an additional option for edge cluster
deployment. Currently, the framework supports only RKE2. By incorporating K3s, a production-grade, CNCF-certified
lightweight and simplified Kubernetes distribution, the framework will allow users to select the most appropriate
Kubernetes distribution for their specific use cases and the resource constraints of their environments. K3s will be
set as the default choice, as it meets the needs of most deployment scenarios with its efficient resource usage, while
RKE2 will remain available for users requiring a full Kubernetes experience for more complex applications. This change
not only enhances flexibility but also lays the groundwork for potentially integrating additional Kubernetes
distributions in the future, further expanding the framework's versatility and adaptability to diverse deployment
scenarios.

## Proposal

### User Stories (Need Update)

#### Story 1

As an edge application deployment engineer, I want to transition from Docker to Kubernetes without incurring
additional overhead for running Kubernetes itself, so that I can allocate more resources to application workloads and
leverage advanced container orchestration capabilities that Kubernetes provides.

#### Story 2

As an edge operator, I want to deploy Kubernetes clusters in environments with limited computational resources, such
as remote IoT installations or small-scale edge sites, and efficiently manage and operate Kubernetes-based
applications without exceeding resource capacities.

#### Story 3

As an edge operator, I want the flexibility to choose between RKE2 and K3s for Kubernetes deployments in specific
environments, allowing me to customize the edge infrastructure to best meet the unique requirements and constraints of
various projects.

#### Story 4

As a system reliability enginner, I want to manage edge clusters with minimal overhead, so that I can focus on
optimizing application performance and reliability.

#### Story 5

As a product manager, I want to ensure that Edge Managibility Framework and Cluster Orchestration is adaptable to
future Kubernetes distributions, so that we can continue to meet evolving user needs and industry standards.

### Design

This section outlines the changes required to integrate K3s as an additional control plane provider within the
Cluster Orchestration. The changes involves updates to the Cluster Orchestration API, controllers, southbound
handlers, sample cluster templates, and the cluster-api-provider-k3s.

#### API Changes

The proposed integration of K3s does not introduce any breaking changes to the existing Cluster Orchestration API, as
defined in the [API spec](https://github.com/open-edge-platform/cluster-manager/blob/release-2.0/api/openapi/openapi.yaml#L904-L909).
Specifically, the current API allows for the specification of `controlplaneprovidertype`, with rke2 being the sole
valid option. (Note that `kubeadm` is listed, but it is intended for internal testing purposes only.) This proposal
will add K3s to the list of valid options and set it as the default control plane provider.

#### Controller Changes

- The Cluster API Provider Intel's controllers are responsible for interacting with the Inventory service to create
and delete workloads, and to reserve and release hosts. The controllers are agnostic as to the flavor of Kubernetes
being run on the host. No changes to the controllers are expected.
- The Cluster Connect Gateway controller is responsible for injecting the connect-agent pod manifest into the static
pod location within Kubernetes. Each Kubernetes distribution has its own default static pod path—for instance, RKE2
uses `/var/lib/rancher/rke2/agent/pod-manifests/`, while K3s uses `/var/lib/rancher/k3s/agent/pod-manifests/`. To
address this, the Cluster Connect Gateway was implemented to support multiple providers. A better approach would be
to add a common standard static pod path, `/etc/kubernetes/manifest`, in the kubelet configuration through the
Cluster Manager, thereby simplifying the Cluster Connect Gateway and ensuring it remains provider-agnostic.

#### Southbound Handler Changes (Open)

The Southbound Handler is responsible for providing cluster bootstrap and cleanup commands to the cluster-agent. The
bootstrap command is essentially a straightforward translation of the cloud-init configuration generated by the CAPI
bootstrap provider, so no additional change is required when integrating a new provider. However, because CAPI
machines are inherently immutable and not assumed reusable, the cluster cleanup command is generated by the
Southbound Handler itself and currently works only for RKE2. This logic should be enhanced to accommodate multiple
providers.

On the other hand, an ongoing discussion is addressing atomic provisioning and cluster creation [add link], with
considerations to disallow machine reuse without re-provisioning. This approach could simplify the Southbound
Handler's implementation by eliminating the need for cleanup command generation, thereby easing the adoption of new
Kubernetes distributions in the future. Additionally, it may remove the necessity for entire Southbound Handler,
allowing cloud-init to handle cluster bootstrapping once cloud-init support is integrated into the Infra Manager,
aligning more closely with CAPI infra provider standards.

#### Sample Cluster Templates

Cluster Orchestration provides sample cluster templates that users can employ to create new clusters directly or as a
basis for developing customized templates with additional configurations. For example, sample templates for RKE2
clusters are available [here](https://github.com/open-edge-platform/cluster-manager/tree/release-2.0/default-cluster-templates).
Similarly, sample cluster templates for K3s, featuring three distinct security levels—restricted, baseline, and
privileged—will be provided and installed by default with the orchestrator.

#### Cluster API Provider K3s Changes

The [K3s CAPI Bootstrap and Control Plane Providers](https://github.com/k3s-io/cluster-api-k3s) enable creating K3s
clusters using the Cluster API framework.  We propose to incorporate these providers into Cluster Orchestration along
side the RKE2 Providers.

We have done a quick review of the K3s Provider code and it is of high quality.  Gaps we found are: (1) concurrent
reconciles are not supported by the controllers, which has been shown to impact scalability with the RKE2 Provider;
(2) the controllers themselves have inadequate unit tests; and (3) updates to the repository are infrequent.

We will contribute a PR to the K3s Provider repo to enable concurrent reconciles.  If we encounter any bugs in the
controllers themselves, we should contribute controller unit tests along with bug fixes.  Finally, since the Cluster
API core controllers are released more frequently than the K3s Provider, it may be necessary to contribute patches in
order to take advantage of new features added to Cluster API.

## Rationale

Alternative approaches, such as MicroK8s and k0s, were considered but ultimately set aside due to specific
limitations. MicroK8s is primarily designed for local development with a focus on single-node setups, which may not
meet the scalability needs of edge deployments. On the other hand, k0s, while suitable for both small and large
clusters, lacks the popularity and community support that K3s enjoys. Here are more details about advantages of K3s
over these alternatives:

**Resource Efficiency**: K3s boasts a compact package size of 68MB (total 556MB) and requires only 1-2 CPUs and
1.7-2GB of RAM, making it particularly well-suited for environments with constrained computational resources. This
efficiency enables more resources to be dedicated to application workloads rather than the Kubernetes itself.

**Ease of Deployment**: As a single binary with embedded data store either SQLite or etcd, K3s simplifies the setup
process and reduces the complexity often associated with Kubernetes deployments.

**Community and Adoption**: With 29.4k GitHub stars, K3s has achieved wide adoption and boasts a strong community,
providing robust support and ongoing development. This popularity underscores its proven track record and reliability
across various deployment scenarios.

**Flexibility and Customization**: K3s offers a variety of default add-ons, such as CNI (flannel by default), helm
controller, and metrics-server, which can be disabled if not required. This flexibility allows users to customize
their deployments to meet specific project requirements and constraints, enhancing adaptability to diverse use cases.

## Affected components and Teams

- Cluster Manager
- Cluster API Provider Intel
- Cluster Tests
- Edge Managibility Framework

## Implementation plan

### Phase 1

The goal for Phase 1 is to ensure successful K3s cluster creation.

Cluster Orch team is responsible for:

- Enhancing the Cluster API provider K3s to production-grade quality by proactively addressing issues we found.
Initiate pull requests promptly to mitigate potential delays from repository maintainers.
- Creating default cluster templates for K3s.
- Modifying Cluster Manager to create custom resources expected by the K3s Provider.
- Modifying Cluster Manager with additional kubelet configuration to make Cluster Connect Gateway provider-agnostic.
- Integrating the K3s Provider with the Cluster API Provider Intel.
- Demonstrating K3s Provider can create / delete clusters in Cluster Tests environment. This may involve manually
creating K3s Provider CRs until Cluster Manager changes are ready.

### Phase 2

The goal for Phase 2 is on integrating K3s with the Edge Manageability Framework (EMF).

Cluster Orch team is responsible for:

- Integrating K3s Provider and other updated components with EMF.
- Ensure non-distruptive upgrade of Cluster Orchestration components.
- Studying cluster creation / deletion behavior across multiple scenarios, and improving where necessary.

### Phase 3

The goal for Phase 3 is to meet specific KPIs, including reducing cluster creation time to under 5 minutes and
scaling to 1k clusters.

Cluster Orch team is responsible for:

- Performing scale testing with K3s edge clusters.
- Optimizing Cluster Orchestration components and Cluster API provider K3s if necessary to achieve the desired KPIs.

### Test Plan

To ensure the reliability and functionality of CO components and the Cluster API K3s provider, it is crucial to
conduct component testing in isolation. This can be accomplished by mocking the Edge Framework Infrastructure Manager
and other dependencies, as we have already done for the current RKE2 providers. This approach allows us to focus on
testing the internal logic of the CO component and verify its expected performance before integrating it with other
system parts.
The integration test plan for Cluster Orch emphasizes testing its subsystems in isolation and reporting the function
test coverage. This plan is hosted, documented, and executed by the
[cluster-tests repository](https://github.com/open-edge-platform/cluster-tests/tree/main). The contents of the
[test plan](https://github.com/open-edge-platform/cluster-tests/tree/main/test-plan) in the cluster-tests repository
includes:

- Test purpose, scope, objective
- Components under test
- Different types of test - functional, failure, error
- Test environment
- Test cases - test id, description, type of test, precondition, steps, expected results
- Test schedule

The cluster-tests repository was designed based on RKE2 providers, so we do not anticipate significant changes when
implementing K3s CAPI providers. This existing framework can be extended to accommodate the current test plan.
Additionally, the repository supports configurable functional integration tests at a granular level, allowing for
adjustments to dependencies as needed.

## Open issues (if applicable)

N/A
