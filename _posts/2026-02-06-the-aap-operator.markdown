---
layout: single
title:  "The Ansible Automation Platform Openshift Operator"
date:   2026-02-06
header:
  og___image: /assets/images/bdt___square.jpeg
---

## Let's set the stage for all of my content

As I build demos and content to share, it's important that I have and easy, repeatable way to demonstrate capabilities that you can use.  I've already covered what the hardware in my homelab looks like and the core infrastructure software I use including the real superhero of modern computing, which is Kubernetes.  In my case, that comes in the form of Red Hat Openshift Container Platform.

The goal of my lab is to represent an enterprise environment as closely as I can, with a "future-now" approach.  What I mean by that is that many of the customers I speak with aren't yet deploying software on Kubernetes or Openshift, however many have been deploying container-based workloads on some flavor of Kubernetes for many years.  For those who are already familiar, what I'm going to show you here will feel "normal."  For those who aren't, I'll do my best to explain some of the details as best I can.

A large amount of the content I share has Ansible as a focus, it makes sense to explain a bit about how I enable the Ansible Automation Platform in my lab.  This starts with the Ansible Automation Platform Operator.

### What is an operator?

To really understand the value the AAP Operator provides it would make sense to start with what an operator is and why they are valuable when running enterprise software on Kubernetes.

The CoreOS Linux development team pioneered the Kubernetes Operator concept in 2016 while searching for a solution to automatically manage container workloads in Kubernetes.  The core concept is to take advantage of the automation capabilities inherent to Kubernetes in order to manage instances of complex applications without the need to customize Kubernetes itself.  The operator's aim is to codify the knowledge that would be attributed to a human who is managing a service or set of services for their organization.  By capturing this operational knowledge as a custom Kubernetes controller and set of one or more custom resources, instances of those services or sets of services can be quickly and easily deployed, configured, and managed without the need to build, maintain, and integrate all of the service's components each and every time.

### What is Ansible and what is Ansible Automation Platform

Over the last several years, I have had numerous conversations about Ansible and more specifically the Ansible Automation Platform.  I think it makes sense to level-set about what both Ansible and AAP are and also what they aren't.

#### Ansible

Ansible is an automation engine.  It has the capability to automate provisioning, installation, configuration, deployment, and management of software, services, and processes.  It uses a modular approach to automation, meaning that anyone can contribute automation capability for a software or hardware product, service, or API.  I like to summarize by saying that if your process, device, or program touches a network or is attached to something which touches a network, it can be automated using Ansible.  There is a vibrant community of Ansible users and contributors, making Ansible one of the most powerful automation tools available today.

![Ansible Overview](/assets/images/2-6-26/1_ansible_architecture.png)

Ansible employs a centralized ___control node___ which acts as an orchestrator.  Ansible is installed only on the ___control node___ and there is no need for an agent or software other than ___python___ (or ___powershell___ for Windows) to be installed elsewhere.  ___Managed nodes___ are defined in a list called ___inventory___ which can contain servers, network devices, workstations, and other endpoints.  Automation processes are defined in ___playbooks___ which are written in ___yaml___ and include a list of ___plays___ with each play containing the parameters for that play and a list of ___tasks___ for the hosts defined in the ___play___.  ___Tasks___ define a ___module___ and other details about individual units of automation work.  ___Modules___ are pre-written, parameterized pieces of automation code, usually written in ___python___ (or ___powershell___ for Windows) which perform specific actions on our ___managed nodes___ such as installing packages, copying data, and deploying configuration templates.  Ansible also employs ___plugins___ which allow for extra non-automation functionality such as privilege escalation and data filtering.  Ansible is executed using a ___command line interface___ or ___CLI___ which reads the defined playbook, connects to the remote ___managed nodes___ and executes the ___module___ code.

#### Ansible Automation Platform

Ansible Automation Platform extends the capabilities of Ansible by enabling IT organizations to create, manage, and scale automation.  This includes multiple additional software components which provide the user interface, APIs, integrations, and capabilities an enterprise needs to maintain control of and visibility into automation across their entire IT landscape.  AAP also includes coding and development tools, certified module, role, and plugin content, tools for organization and curation of custom and collected content, analytics tools, and event-driven capabilities, making it a platform for much more than just automation.

![AAP Components](/assets/images/2-6-26/2_aap_upstream.png)

Many IT professionals who are familiar with Ansible know about the AWX project, also known as ___Automation Controller___ (previously known as ___Ansible Tower___).  At one time, Ansible Automation Platform consisted only of the API and UI that made up AWX/Tower.  Now, there are over 24 open-source projects which make up the platform, including AWX, galaxy___ng, eda-server, and much, much more.  All of these upstream projects have been integrated into one single, supported platform.

Long story short, Ansible Automation Platform is much more than AWX or Ansible Tower.

### The Ansible Automation Platform Operator

The additional components of Ansible Automation Platform make it an excellent candidate for delivery as Kubernetes operator.

![Operator Details](/assets/images/2-6-26/3_aap_operator_details.png)

The operator includes 6 controller-managers which get deployed in an Openshift cluster.  Those controller-managers continuously evaluate either the entire cluster, or a specific namespace, ensuring that any custom resources created via the operator's custom resource definitions remains in the configuration which was defined when it was deployed.

![Operator Controller Managers](/assets/images/2-6-26/4_aap_operator_controller_managers.png)

There are a multitude custom resource definitions which can be used to deploy components of Ansible Automation Platform, either as a fully integrated, complete platform, or as individual components.  Additionally, there are backup and restore capabilities as well as resource deployment options included.  This means disaster recovery solutions are built into the operator itself and individual automation resources can be deployed into Ansible Automation Platform itself.  This adds the additional benefit of making Configuration as Code via modern CI/CD or GitOps tools a real possibility, right out of the box.  The CRDs can be used to deploy all of the necessary Inventories, Credentials, Projects, Job Templates, and Workflows for your automation needs, all via deployment pipelines as you would for any other application.

![Operator CRDs](/assets/images/2-6-26/5_aap_operator_crds.png)

### Installing the AAP Operator and Configuring an AAP Instance

Installing the Ansible Automation Platform operator is straightforward.  There are some prerequisites and recommendations.

- You must be running Red Hat Openshift Container Platform
  - This can be any Openshift deployment including managed offerings on Azure or AWS
  - AAP subscriptions ___include___ self-managed Openshift for the purposes of running AAP
- ReadWriteMany Storage is recommended when deploying Automation Hub
- Care should be taken in setting Request and Limit ranges
  - This will be dependent on your automation workloads

I'm only going to cover the command-line instructions here.  For instructions regarding the UI, refer to the official [Red Hat Documentation](https://docs.redhat.com)

#### Installing the AAP operator via the command line

1. To install the AAP operator, you first need to log in to your cluster with appropriate privileges and create a project/namespace.

    ```shell
    oc login -u <username> https://api.<cluster_fqdn>:6443
    ```

    ```shell
    oc new-project ansible-automation-platform
    ```

2. Then you need to create a subscription manifest to subscribe to the Ansible Automation Platform operator.  You can name this file something like _sub.yaml_

    ```yaml
    apiVersion: operators.coreos.com/v1
    kind: OperatorGroup
    metadata:
      name: ansible-automation-platform-operator
      namespace: ansible-automation-platform
    spec:
      targetNamespaces:
        - ansible-automation-platform
    ---
    apiVersion: operators.coreos.com/v1alpha1
    kind: Subscription
    metadata:
      name: ansible-automation-platform
      namespace: ansible-automation-platform
    spec:
      channel: 'stable-2.6'
      installPlanApproval: Automatic
      name: ansible-automation-platform-operator
      source: redhat-operators
      sourceNamespace: openshift-marketplace
    ```

    This subscription manifest will create a subscription called ___ansible-automation-platform___ which subscribes the ___ansible-automation-platform___ namespace to the ___ansible-automation-platform-operator___ operator.

3. Run `oc apply` to apply the subscription

    ```shell
    oc apply -f sub.yaml
    ```

4. Verify that the CSV PHASE reports "Succeeded" before proceeding.

    ```shell
    oc get csv -n ansible-automation-platform

    NAME                               DISPLAY                       VERSION              REPLACES                           PHASE
    aap-operator.v2.6.0-0.1728520175   Ansible Automation Platform   2.6.0+0.1728520175   aap-operator.v2.6.0-0.1727875185   Succeeded
    ```

    Succeeded means that the operator was installed successfully and the controller-managers are evaluating the ___ansible-automation-platform___ namespace for any custom resources deployed via the operator.

#### Configuring an AAP Instance

Now that the operator is installed, we can use the ___AnsibleAutomationPlatform___ custom resource definition to deploy a complete instance of AAP, including the Platform Gateway, Automation Controller, Automation Hub, and Event-Driven Ansible Controller.  This one manifest will deploy all of the components of the platform, the necessary database(s), redis, and any dependent storage required for Ansible Automation Platform.

1. Create a manifest for your AAP instance.  You can name this file something like ___aap.yaml___

    ```yaml
    apiVersion: aap.ansible.com/v1alpha1
    kind: AnsibleAutomationPlatform
    metadata:
      name: example
      namespace: ansible-automation-platform
    spec:
      image_pull_policy: IfNotPresent
      controller:
        disabled: false
      eda:
        disabled: false
      hub:
        disabled: false
        file_storage_size: 500Gi
        file_storage_storage_class: <your-read-write-many-storage-class>
      lightspeed:
        disabled: true
    ```

    This manifest specifically disables the Ansible Lightspeed components and configures a relatively large storage volume for Automation Hub.

2. Run `oc apply` to deploy the Ansible Automation Platform instance

    ```shell
    oc apply -f aap.yaml
    ```

3. Now you can watch as all of the pods are started for your shiny, new Ansible Automation Platform installation.

    ```shell
    watch "oc get pods -n ansible-automation-platform"
    ```

    ```shell
    Every 2.0s: oc get pods | grep example

    example-controller-migration-4.7.8-zd4q6                          0/1     Completed   0               3m16s
    example-controller-task-7c75565d6c-kr5c4                          4/4     Running     0               3m20s
    example-controller-web-7868bd8f87-nj9q6                           3/3     Running     0               3m22s
    example-eda-activation-worker-5b658fc85c-m7jt2                    1/1     Running     0               3m35s
    example-eda-activation-worker-5b658fc85c-vwqnz                    1/1     Running     0               3m35s
    ...
    ...
    ```

4. When the platform has completed installation, you can find the route to access the AAP instance.

    ```shell
    oc get route
    NAME                    HOST/PORT                                   PATH   SERVICES                        PORT   TERMINATION     WILDCARD
    example              example-aap.apps.ocp.bk.lab                     example                      http   edge/Redirect   None
    example-controller   example-controller-aap.apps.ocp.bk.lab          example-controller-service   http   edge/Redirect   None
    example-eda          example-eda-aap.apps.ocp.bk.lab                 example-eda-api              8000   edge/Redirect   None
    example-hub          example-hub-aap.apps.ocp.bk.lab          /      example-hub-web-svc          8080   edge/Redirect   None
    ```

5. Extract the admin password from the ___example-admin-password___ secret

    ```shell
    oc get secret example-admin-password --template={% raw %}"{{.data.password|base64decode}}"{% endraw %}
    ```

6. Find the route to your instance.  It is the one in the list which matches the name of the instance you deployed, in this case the top one in the list.

    ```shell
    oc get route
    NAME                    HOST/PORT                                   PATH   SERVICES                        PORT   TERMINATION     WILDCARD
    example                 example-aap.apps.ocp.bk.lab                        example                         http   edge/Redirect   None
    example-controller      example-controller-aap.apps.ocp.bk.lab             example-controller-service      http   edge/Redirect   None
    example-eda             example-eda-aap.apps.ocp.bk.lab                    example-eda-api                 8000   edge/Redirect   None
    example-hub             example-hub-aap.apps.ocp.bk.lab             /      example-hub-web-svc             8080   edge/Redirect   None
    ```

7. Browse to the url in a web browser, in this case ___example-aap.apps.ocp.bk.lab___ and log in using the username `admin` and the password extracted from step 5

    ![AAP Login Page](/assets/images/2-6-26/6_aap_login_page.png)

8. From here you will need to attach a valid Ansible Automation Platform subscription.  I'm not covering that here.  If you want to try out AAP, or have AAP subscriptions and you don't know how to get them, reach out to your Red Hat account team.

    ![AAP Subscription Page](/assets/images/2-6-26/7_aap_subscription_page.png)

### Final Thoughts

The Ansible Automation Platform Operator for Openshift is an easy and efficient way to enable a production ready Ansible Automation Platform installation for all of your enterprise automation needs, especially in cases where you already have Openshift up and running.

For those without Openshift, I will follow up with a post regarding the Containerized installation method, which runs on Red Hat Enterprise Linux.  It still utilizes containers, but does not require the overhead of also running Openshift.