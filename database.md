# It is a Database

Kubernetes is a database. Like any other database tech, it provides an API that wraps a structured persistent storage engine called etcd with specific semantics. Those semantics allow users or other components to create, read, update, and delete (CRUD) arbitrary structured data.

The Kubernetes API is REST on HTTP(S), but most people will use a command line tool to interact with that API. The tool is called `kubectl` and is installed and configured by Docker for Desktop automatically when you enable Kubernetes.

Kubernetes, like other databases, is all about working with records in schemas. What a relational database might call a table is called a Kind, and "rows" are called "resources" in Kubernetes.

Most Kubernetes distributions include several Kinds by default. Those are used to describe all sorts of operational resources that you typically encounter in a service software environment. For example Kubernetes includes: Job, CronJob, Service, Deployment, Event, NetworkPolicies, Secrets, and ConfigMaps. But Kubernetes is better suited for storing operational data than most other database solutions.

* Resource schemas are self-documenting. (`kubectl explain <resource>`)
* API semantics and mechanisms support extreme extensibility and integration with distributed state management best-practices.
* The API supports scalable event-driven "trigger-like" behavior.
* Powerful role-based access control is included by default.
* Encapsulation is only implemented via access control making all Kubernetes-homed data simple to automate against.

Before jumping into automation or workflow resources take some time to use the more direct database functionality.

## Exercises

### 1. Verify your Kubernetes version

The `kubectl` command provides a `version` subcommand. By default its output is a long string of versions and difficult to read. Use the `-o yaml` option to get a nice structured response. Most `kubectl` commands are actually just API wrappers and you can use the `-o` flag to have raw YAML or JSON responses printed to the terminal.

Run `kubectl version -o yaml` and verify that the client and server are version 1.10 or later. Ideally the client and server version will match.

If you get an error verify that Kubernetes is running. If you still get an error check out the troubleshooting guide on Kubernetes's website.

### 2. Create a Custom Schema (Kind)

The following are the contents of [business-papers.yaml](database/business-papers.yaml). It defines a Custom Resource Definition (CRD) for a new kind called Paper (or Papers).

```yaml
apiVersion: apiextensions.k8s.io/v1beta1    # Tells Kubernetes what API version of this resource (the CRD)
kind: CustomResourceDefinition              # Tells Kubernetes that this is a CRD - just another kind of data
metadata:
  name: papers.workshop.gotopple.com        # The name of this CRD

# All Kubernetes resources have a "spec." Think of this like the data or content of the resource.
spec:
  group: workshop.gotopple.com              # CRDs have a validated group attribute
  names:
    kind: Paper                             # The CRD that this document defines should be named "Paper"
    plural: papers                          #   or "papers."
  scope: Namespaced                         # Means we can have different CRDs named "Paper" in different 
                                            #   namespaces without conflict.
  version: v1alpha1                         # The CRD defined by this spec is version, "v1alpha1"
```

A CRD is a resource that tells Kubernetes about a Kind of resource. Think of it like a table definition in a database. You might notice that this CRD only gives the new kind a name (and group, and version), and doesn't specify any columns or properties. Kubernetes is a document database and document structures are constrained by validation rules. This CRD does not provide any validation rules.

Add this CRD to your cluster using the command: 

```sh
kubectl create -f database/business-papers.yaml
```

You should see output like: 

```
customresourcedefinition.apiextensions.k8s.io "papers.workshop.gotopple.com" created
```

You can list all of the CRDs installed in your cluster by running `kubectl get crd`. And once the CRD has been installed it will show up in that list and you'll be able to operate on Paper resources. For example, you can get a list of Papers by running `kubectl get paper`.

When you do so you'll notice that the list is empty. Next create a new paper.

### 3. Create Resources (Papers)

You will find two Papers in the databases folder as well. Open the first example in [example-paper.yaml](database/example-paper.yaml). There are only a handful of expected properties for resources. Every resource will have an `apiVersion`, a `kind`, a `metadata` section that includes a `name`, and a `spec`. The `apiversion` and `kind` must match an existing Kind in your cluster. Everything under `spec` is constrained by the CRD validation (which does not exist in this example).

```yaml
apiVersion: workshop.gotopple.com/v1alpha1
kind: Paper
metadata:
  name: first-paper
spec:
        author: Jeff
        type: memo
        content: >
          Dear reader,

          This is an example memo demonstrating how we can use Kubernetes to 
          store arbitrary information. The Papers kind will have already been
          created, but in this case it does not have any validation rules. 

          Kubernetes uses validation rules to enforce document structures.
          Without them a user can store any data they want in a resource.

          Run "kubectl create -f database/example-paper.yaml" to add this file
          to your local cluster and then run, "kubectl get papers" to see
          "first-paper" added to the list. Run 
          "kubectl get papers first-paper -o yaml" to view the the raw resource
          in YAML form.

          Enjoy,
          Jeff
```

Create the Paper in your cluster:

```sh
kubectl create -f database/example-paper.yaml
```

You should get output like:

```
paper.workshop.gotopple.com "first-paper" created
```

Now list and retrieve the Paper from your cluster. Run:

```sh
kubectl get papers
kubectl get paper first-paper
```

Retrieve the raw YAML for the first-paper:

```sh
kubectl get paper first-paper -o yaml
```

At this point you've created your first record in a custom schema in your personal Kubernetes database. Okay, well, technically two records... One for the CRD and one for the Paper. Things get more interesting when you start adding multiple resources. Add a second one from [second-example-paper.yaml](database/second-example-paper.yaml): `kubectl create -f database/second-example-paper.yaml`.

That resource looks like:

```yaml
apiVersion: workshop.gotopple.com/v1alpha1
kind: Paper
metadata:
  name: second-paper
spec:
        writer: Portia
        kind: memo
        data: >
          Dear reader,

          This is an entirely different document schema. This document uses "writer"
          instead of "author," "kind" instead of "type," and "data" in place of 
          "content." You are able to create both the "first-paper" and "second-paper" 
          resources as the same Kind because there are no validators on the type.

          Enjoy,
          Portia
```

Now you can get both resources from Kubernetes at the same time and in a cohesive stream:

```sh
kubectl get papers -o yaml
```

The result is a single document where each Paper is included as an item in the `items` list:

```yaml
apiVersion: v1
items:
- apiVersion: workshop.gotopple.com/v1alpha1
  kind: Paper
  metadata:
    clusterName: ""
    creationTimestamp: 2018-09-27T17:55:19Z
    generation: 1
    name: first-paper
    namespace: default
    resourceVersion: "493415"
    selfLink: /apis/workshop.gotopple.com/v1alpha1/namespaces/default/papers/first-paper
    uid: 7f6197ed-c27e-11e8-85dd-025000000001
  spec:
    author: Jeff
    content: "Dear reader,\nThis is an example memo demonstrating how we can use Kubernetes
      to  store arbitrary information. The Papers kind will have already been created,
      but in this case it does not have any validation rules. \nKubernetes uses validation
      rules to enforce document structures. Without them a user can store any data
      they want in a resource.\nRun \"kubectl create -f database/example-paper.yaml\"
      to add this file to your local cluster and then run, \"kubectl get papers\"
      to see \"first-paper\" added to the list. Run  \"kubectl get papers first-paper
      -o yaml\" to view the the raw resource in YAML form.\nEnjoy, Jeff\n"
    type: memo
- apiVersion: workshop.gotopple.com/v1alpha1
  kind: Paper
  metadata:
    clusterName: ""
    creationTimestamp: 2018-09-27T18:09:23Z
    generation: 1
    name: second-paper
    namespace: default
    resourceVersion: "494337"
    selfLink: /apis/workshop.gotopple.com/v1alpha1/namespaces/default/papers/second-paper
    uid: 766224e7-c280-11e8-85dd-025000000001
  spec:
    data: |
      Dear reader,
      This is an entirely different document schema. This document uses "writer" instead of "author," "kind" instead of "type," and "data" in place of  "content." You are able to create both the "first-paper" and "second-paper"  resources as the same Kind because there are no validators on the type.
      Enjoy, Portia
    kind: memo
    writer: Portia
kind: List
metadata:
  resourceVersion: ""
  selfLink: ""
```

### 4. Querying the Database (selectors)

While Kubernetes does not enable you to query based on arbitrary document properties, it does index and provides query interfaces on document taxonomy. Specifically, every resource can have **Labels** added to its `metadata` property. Labels are arbitrary key-value string pairs.

Neither of the Papers that you added have any labels applied. So, to make this exercise a bit more interesting add a label to the `first-paper` paper using the following command:

```sh
kubectl label paper first-paper author=jeff type=memo
kubectl label paper second-paper author=portia type=memo
```

In this case you added two labels to each resource (you could have done this by modifying the YAML as well). You specifically labeled an author and a type. However, you could have added any labels that make sense. Generally you want labels to represent some cross-cutting property, not always something that is in the document itself. But this is a happy example.

After the resources are labeled you can query based on those labels. Get a list of all of the papers authored by Jeff: 

```sh
kubectl get paper -l author=jeff
``` 

Then get all of the papers that are memos in raw YAML: 

```sh
kubectl get paper -l type=memo -o yaml
```

In the second case you can see where the labels were added to the `metadata` for each resource. These demonstrate all of the basic principals and tools for working with data in a Kubernetes cluster. Working with Every resource type (be it a workload resource, a network resource, or an RBAC resource) will feel similar and you'll use similar patterns for working with the data. Kubectl does expose other higher-level or imperative operations, but know that those are doing the same things under the covers.

Next up you'll get started with basic Kubernetes [workload resources](workload.md).
