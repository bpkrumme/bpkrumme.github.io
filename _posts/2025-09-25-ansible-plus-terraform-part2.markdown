---
layout: single
title:  "Simple Cloud Provisioning and Automation with Terraform and Ansible - Part 2: Orchestrating Terraform with Ansible"
date:   2025-09-25
header:
  og_image: /assets/images/bdt_square.jpeg
---

## There's still more than one best tool...

In Part 1, I showed you how to configure a basic Terraform project and deploy some infrastructure, in particular EC2 instances on AWS.  Terraform is a best of breed Infrastructure-as-Code utility and really excels at provisioning the infrastructure components I need to enable a new IT service.  For Day 0 Operations, Terraform is an excellent choice.

I will admit that I am a bit biased when it comes to Ansible, and especially Ansible Automation Platform.  It is, of course, the Red Hat product I specialize in as a pre-sales Solution Architect.  It is my opinion, regardless of that bias, that Ansible should remain at the center of the IT operations landscape for one very important reason:  It integrates easily with just about every other technology in use.  This includes ITSM tools like Servicenow, observability utilities like Dynatrace or BigPanda, all of the major cloud providers, Linux and Windows servers, and a multitude of industry standard software solutions.

Let's explore Ansible and its strengths, and recap those it shares with Terraform.

### Ansible Strengths

* Automation and Orchestration:  Ansible is primarily an automation engine.  It has the capability to be the centralized orchestration hub for all manners of IT operations, provisioning, deployment, configuration, and lifecycle.
* Ease of Use: Ansible is known for its simplicity and ease of use. It uses a human-readable scripting language (YAML) for playbooks, making it easy to understand and write.
* Agentless Architecture: Ansible does not require any agents on the managed nodes. It uses SSH (or WinRM) for communication, which makes it more secure and easier to set up.
* Idempotent: Ansible plays are idempotent, meaning they can be run multiple times without causing any unwanted side effects.

### Shared Strengths with Terraform

* Community and Ecosystem: Both Ansible and Terraform have large and active communities, with many third-party modules and tools available.
* Integrations: Both Ansible and Terraform can integrate with other tools such as with CI/CD pipelines.
* Modularity and Reusability: Terraform and Ansible both allow you to create modular and reusable code, making it easier to manage complex infrastructure deployments and their configurations.

As said previously, the tools complement each other nicely.  

## Orchestration of Provisioning with Ansible

Now that I have a Terraform project, I need an Ansible project which I will use to apply the Terraform configuration.  This might seem like I'm just introducing another tool because I like it, but there's a very important reason for wrapping the Terraform with Ansible.  I have other configuration and deployment I want to do later...and I have Ansible Automation Platform I'm already using for that configuration and deployment of my on-premises infrastructure.  Instead of reinventing the wheel, I want to use the automation jobs I already have to also configure my cloud instances.

### Prerequisites

I need to start by creating an Ansible project that can apply Terraform configurations.  Like with Terraform, there are some tools and configurations I need in place before I can begin the orchestration journey with Ansible.  Here's what I need for Ansible:

Necessary Tools:

* ansible-core
* The cloud.terraform Ansible Collection
* A text editor

Optional Tools and Add-Ons:

* An IDE (Integrated Development Environment) such as Visual Studio Code
* The Ansible extension for VS Code
* Git or other source control
* Ansible Development Tools such as Ansible Navigator and Ansible Builder

Again, my IDE of choice is Visual Studio Code with several useful extensions installed including the Ansible, Remote Development, GitLens, and Code Spell Checker extensions to name a few.

### A basic Ansible Project for Automating Terraform

Ansible projects can be very simple or very complex.  In the simplest of forms, I could have just a playbook which defines the automation I want to run.  I could also have Ansible roles or collections I've created which standardize the way I apply certain configurations, and include extra configurations for Ansible Navigator, Molecule, and other Ansible-related development tools.

Here's what my simple Ansible project for orchestrating Terraform looks like:

```text
cloud_ops
├── ansible-navigator.yml
├── playbooks
│   ├── terraform_apply.yml
│   └── terraform_destroy.yml
```

#### ansible-navigator.yml

I use Ansible Navigator almost exclusively when developing Ansible projects.  The primary reason is because I can use an Execution Environment, even when developing my playbook projects in a Linux shell.  This is important because Automation Controller ***ONLY*** uses Execution environments when launching automation jobs, so I can ensure my automations run the same on my local development workstation as they will in Automation Controller.

There will be a future post about building custom Execution Environments.  For now, you can find more about Execution Environments here:  [Getting started with Execution Environments](https://docs.ansible.com/ansible/latest/getting_started_ee/index.html)

Let's review the contents of my `ansible-navigator.yml` configuration file:

##### logging configuration

First things first, I want to make sure that the logs generated by Ansible Navigator go to a directory which I can then exclude from Git tracking.  This helps ensure that I'm not committing unnecessary data to source control, and it keeps things tidy in my project directory.

```yaml
  logging:
    level: debug
    append: false
    file: $PWD/.logs/ansible-navigator.log
```

##### playbook-artifact configuration

When I run a playbook using Ansible Navigator, a JSON-formatted `artifact` file is generated.  This serves as a way to look back at previous playbook runs, similar to the logging mechanism of Automation Controller, but in my local environment instead of a GUI.  It is especially useful when I are developing complex automation using collections and roles, which I need to debug before committing my code.  I also want to exclude these from Git tracking, so I put them in the same place as my logs.

```yaml
  playbook-artifact:
    enable: true
    save-as: $PWD/.logs/{playbook_name}-artifact-{time_stamp}.json
```

##### execution-environment configuration

Finally, I configure the default Execution Environment for this project.  I want to minimize the amount of typing and the complexity of commands I use on the command line, so setting the default here will ensure I have the correct Execution Environment selected when I launch playbooks.  I can override this if necessary, however that is usually not needed within the confines of one project.

```yaml
  execution-environment:
    container-engine: auto
    enabled: true
    pull:
      policy: missing
    image: production-aap.apps.ocp.bk.lab/homelab_ees/cloud_ee:latest
```

A couple of things to point out here.  First, I could disable using an Execution Environment by setting `enabled: false`.  Second, notice that I have the pull policy set to `missing`.  This means that Ansible Navigator will ONLY try to pull the EE container image if it is not currently cached on my workstation.  I also have the image tag of `:latest` configured.  Those two settings combined mean that it will check for the latest tagged version of the Execution Environment whenever I run a playbook with Ansible Navigator and pull the image if necessary.  If I have the current latest image cached, it will continue without pulling the image.

I am using a custom Execution Environment.  Should you want to build the EE yourself, you can find the configuration here:  [Cloud EE Configuration](https://github.com/bpkrumme/ansible_ees/tree/main/cloud_ee)

##### Complete Ansible Navigator Configuration

Here is my complete configuration for reference:

```yaml
---
ansible-navigator:
  logging:
    level: debug
    append: false
    file: $PWD/.logs/ansible-navigator.log

  playbook-artifact:
    enable: true
    save-as: $PWD/.logs/{playbook_name}-artifact-{time_stamp}.json

  execution-environment:
    container-engine: auto
    enabled: true
    pull:
      policy: missing
    image: production-aap.apps.ocp.bk.lab/homelab_ees/cloud_ee:latest
```

#### The Playbooks

Now that I have a good Ansible Navigator configuration and Execution Environment, I can actually start automating the Terraform configuration using builtin modules and the cloud.terraform Ansible Collection.  

Ansible playbooks are quite simple to understand.  Because they are written in YAML, they are mostly human-readable.  I won't cover writing playbooks as this is very well documented.  Check out that documentation here:  [Creating a Playbook](https://docs.ansible.com/ansible/latest/getting_started/get_started_playbook.html)

In my project I have a `playbooks` directory which is where all playbooks go.  There are two in this project that I will explain.

##### terraform_apply.yml

My `terraform_apply.yml` playbook includes only one play called `Apply Terraform Plan from a Git Repository` which I run against `localhost`.  I don't need a managed node in this case, because it's simply going to sync my Terraform git repository and apply the configuration.  Because I'm using an Execution Environment, `localhost` actually means the container which is launched, using the EE container image and NOT my workstation shell environment.  This means I have to ensure some data gets passed into the EE when launched.

I have also disabled privilege escalation using the `become` keyword with a value of `false`.  This is because I don't need to run Terraform as a privileged user.

There are two tasks defined in the play:

First, I synchronize the Terraform project git repository to the containerized Ansible environment using the `ansible.builtin.git` module.  Because I want this playbook to be reusable, the repository namespace and name are set using the `terraform_namespace` and `terraform_project` variables.  I'll need to set these variables when I launch the playbook, otherwise it will fail.  I also use the `terraform_project` variable to determine the git clone destination directory.

The second task runs the Terraform plan from the synchronized repository using the `cloud.terraform.terraform` module.  I use the same directory for the `project_path` parameter as was set in the `ansible.builtin.git` module's `dest` parameter.  Setting the `state` parameter to `present` ensures that I apply the configuration as defined in the Terraform project.  It will have the same affect as running `terraform apply` on the CLI.  I also have the `force_init` parameter set to `true` which is necessary to initialize the Terraform environment, just like running `terraform init` in my Execution Environment before applying the configuration.  Without `force_init` set, the plan would fail as the necessary Terraform providers would not be present in my environment.

Here is the entire `terraform_apply.yml` playbook:

```yaml
---
- name: Apply Terraform Plan from a Git Repository
  hosts: localhost
  become: false
  tasks:
    - name: Sync the terraform project directory
      ansible.builtin.git:
        repo: https://github.com/{{ terraform_namespace }}/{{ terraform_project }}
        dest: /tmp/{{ terraform_project }}
    - name: Run terraform apply
      cloud.terraform.terraform:
        project_path: /tmp/{{ terraform_project }}
        state: present
        force_init: true
```

##### terraform_destroy.yml

Because I might want to also destroy the infrastructure created by Terraform, I've also created a `terraform_destroy.yml` playbook.  It is defined and works nearly identically to the `terraform_apply.yml` playbook, but instead of ensuring that the Terraform configuration is present I have set the `state` parameter to `absent`.  This is going to have the same affect as running `terraform destroy` on the CLI.

Here is the complete `terraform_destroy.yml` playbook:

```yaml
---
- name: Destroy Terraform Plan from a Git Repository
  hosts: localhost
  become: false
  tasks:
    - name: Sync the terraform project directory
      ansible.builtin.git:
        repo: https://github.com/{{ terraform_namespace }}/{{ terraform_project }}
        dest: /tmp/{{ terraform_project }}
    - name: Run terraform destroy
      cloud.terraform.terraform:
        project_path: /tmp/{{ terraform_project }}
        state: absent
        force_init: true
```

### Executing the Automation

Now that I have an Ansible project defined, it's time to launch the automation and test the playbooks to make sure they're working as I expect.  Here's how to do that:

1. Ensure the Execution Environment (if using one) includes all of the prerequisites for Terraform including the Terraform CLI and Cloud Provider CLI

2. Ensure the Execution Environment includes the `cloud.terraform` Ansible Collection

3. Launch the playbook, ensuring that cloud provider credentials are set, if necessary, as well as the playbook variables.  Because I am deploying to AWS, I need to set the `AWS_SECRET_ACCESS_KEY` and `AWS_ACCESS_KEY_ID` environment variables using the `--senv` or `--set-environment-variable` CLI option for ansible-navigator.  Additionally, I need to set the `terraform_namespace` and `terraform_project` variables using the `-e` or `--extra-vars` CLI option.

    Ansible Navigator has two options for output.  You can use the text-based user interface, OR use standard output.  I like standard output, so I'll use the `-m stdout` CLI option as well.

    Here's what the full command and output might look like:

    ```shell
    $ ansible-navigator run playbooks/terraform_apply.yml --senv AWS_ACCESS_KEY_ID=<your access id>--senv AWS_SECRET_ACCESS_KEY=<your secret key> -e "terraform_namespace=bpkrumme terraform_project=terraform_basic" -m stdout`
    ```

    ```text
    PLAY [Apply Terraform Plan from a Git Repository] ******************************

    TASK [Gathering Facts] *********************************************************
    ok: [localhost]

    TASK [Sync the terraform project directory] ************************************
    changed: [localhost]

    TASK [Run terraform apply] *****************************************************
    changed: [localhost]

    PLAY RECAP *********************************************************************
    localhost                  : ok=3    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
    ```

4. Should you want to destroy the infrastructure you just created, you can use the exact same syntax to run the `terraform_destroy.yml` playbook:

    ```shell
    $ ansible-navigator run playbooks/terraform_destroy.yml --senv AWS_ACCESS_KEY_ID=<your access id>--senv AWS_SECRET_ACCESS_KEY=<your secret key> -e "terraform_namespace=bpkrumme terraform_project=terraform_basic" -m stdout
    ```

    ```text
    PLAY [Destroy Terraform Plan from a Git Repository] ****************************

    TASK [Gathering Facts] *********************************************************
    ok: [localhost]

    TASK [Sync the terraform project directory] ************************************
    changed: [localhost]

    TASK [Run terraform destroy] ***************************************************
    changed: [localhost]

    PLAY RECAP *********************************************************************
    localhost                  : ok=3    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
    ```

### Ansible Documentation and Resources

I understand that my examples here may not be 100% complete, and I don't want you fumbling through your first attempts at running an Ansible project.  Here are some resources to help.

#### Documentation

The Ansible documentation is excellent and includes everything from basic to advanced Ansible concepts.  Check it out here:  [Ansible Documentation](https://docs.ansible.com)

#### My Example Project

If you would like to use my basic project as a reference you can find the full configuration here:  [Project source on GitHub](https://github.com/bpkrumme/cloud_ops)

### Up Next

Be on the lookout for part 3, where I will create a workflow in Automation Controller to apply the Terraform and also configure the EC2 instances.
