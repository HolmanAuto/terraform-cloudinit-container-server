# Deploy an application to any Cloud™ VM with Terraform, Docker & cloud-init

[![GitHub tag (latest SemVer)](https://img.shields.io/github/v/tag/christippett/terraform-cloudinit-container-server?label=Version)](./CHANGELOG.md) [![Terraform Registry](https://img.shields.io/badge/Terraform-Registry-623CE4)](https://registry.terraform.io/modules/christippett/container-server/cloudinit/)

This Terraform module enables you to quickly and easily deploy a single container or `docker-compose.yaml` manifest to an instance running on any of the major cloud platforms.

No external dependencies.

No proprietary frameworks.

Just plain ol' `docker`, `docker-compose` and `systemd` — deployed with `cloud-init` using a single, cloud-agnostic configuration script.

# What does it do?

- ☁️ Quickly deploy a single container image or `docker-compose.yaml` manifest to a VM running on any of the following clouds:
  - **AWS** ([see example](./examples/aws-docker-image-simple/))
  - **Google Cloud Platform** ([see example](./examples/gcp-docker-image-simple/))
  - **DigitalOcean** ([see example](./examples/digitalocean-docker-image-simple/))
  - **Azure** ([see example](./examples/azure-docker-image-simple))
  - _...and theoretically any other vendor that supports cloud-init_
- 🌐 Host multiple services on the same domain, with routing and service discovery provided by **Traefik**.
- 🔑 Automatic SSL/TLS certificates generated by **Let's Encrypt**.
- 🏗 Easy to configure via **Terraform**, with the resulting configuration files rendered from templates that can be easily extended or overriden.

# What's this cloud-init thing?

This whole project started as an experiment into how `cloud-init` could be used to easily bootstrap and configure instances across clouds with little to no refactoring effort or vendor-specific allowances being made.

I'll defer to Canonical for the full pitch:

> _Cloud-init is the industry standard multi-distribution method for cross-platform cloud instance initialization. It is supported across all major public cloud providers, provisioning systems for private cloud infrastructure, and bare-metal installations._
>
> _Cloud-init will identify the cloud it is running on during boot, read any provided metadata from the cloud and initialize the system accordingly. This may involve setting up network and storage devices to configuring SSH access key and many other aspects of a system. Later on cloud-init will also parse and process any optional user or vendor data that was passed to the instance._
>
> Source: https://github.com/canonical/cloud-init

# What else do I need to know?

Whichever cloud platform you decide to use, it's important to pick an appropriate base OS image for your VM that supports both `docker` and `systemd`. Everything else is installed as a container. The following operating systems have been tested to work successfully:

- [Google's Container Optimized OS](https://cloud.google.com/container-optimized-os)
- [Amazon Linux 2](https://aws.amazon.com/amazon-linux-2/)
- [Ubuntu 18.04 (via DigitalOcean's Marketplace)](https://marketplace.digitalocean.com/apps/docker)

# Why use this over something like Fargate or Cloud Run?

Even the most basic and cheapest of VMs are capable of running _a lot_ of containers. As fantastic as the cloud's PaaS and serverless offerings are, it's sometimes easier to orchestrate several containers on one machine without having to mess around with IAM, networking, service inter-dependencies etc. Deploying several services with Docker Compose can be a cost-effective and simpler alternative when hosting small hobby projects, POCs or other experimental workloads.

To throw about some quick numbers, below are the going rates for a low-cost VM running on each of the major cloud platforms. These would be more than capable of running dozens of containers, especially if you don't expect them to receive much traffic.

- **AWS**
  - **Cost:** USD\$4.76/month<br />
    _t3a.micro • 2vCPU/1GB • 10GB HDD_
- **Google Cloud Platform**
  - **Cost:** USD\$6.11/month\*\*<br />
    _e2.micro • 0.25vCPU/1GB • 10GB HDD_
- **DigitalOcean**
  - **Cost:** USD\$6.00/month\*\*<br />
    _Standard Droplet • 1vCPU/1GB • 10 HDD_
- **Azure**
  - **Cost:** USD\$14.73/month\*\*<br />
    _A0 • 1vCPU/0.75GB • 32GB HDD_

# Getting started

The output from this module is a string containing the cloud-init configuration required to setup and deploy/run your container(s) on a virtual machine. Reference the output from this module when defining your instance's `user_data` (AWS / DigitalOcean), `metadata.user-data` (Google Cloud) or `custom_data` (Azure). When the instance is created, cloud-init will trigger and do its thing.

Some cloud Terraform resources expect the content to first be base64 encoded (e.g. Azure's `azurerm_linux_virtual_machine`), refer to the Terraform documentation below for details relevant to the cloud provider you're using:

- [AWS Documentation](https://www.terraform.io/docs/providers/aws/r/instance.html#user_data)
- [Google Cloud Documentation](https://www.terraform.io/docs/providers/google/r/compute_instance.html#metadata)
- [Azure Documentation](https://www.terraform.io/docs/providers/azurerm/r/linux_virtual_machine.html#custom_data)
- [DigitalOcean Documentation](https://www.terraform.io/docs/providers/do/r/droplet.html#user_data)

## Example 1: Deploying a single container image

The fastest way to get started is to deploy a single image. All you need to specify is the image you want to deploy. The module's `container` input variable accepts any attribute normally found under Docker Compose's `service` key ([docs](https://docs.docker.com/compose/compose-file/#service-configuration-reference)). You may find that `image` and `ports` are all you need to define to get your container up and running.

Other than the `container`, the other variables we need to provide values for are `domain` and `email`, these are needed for Let's Encrypt to provision an SSL certificate for your domain.

Don't want to trip the rate limit for Let's Encrypt while you're still getting things set up? Set `letsencrypt_staging` to `True` until you're confident things are working just right.

```hcl
module "container-server" {
  source  = "christippett/container-server/cloudinit"
  version = "~> 1.2"

  domain = "example.com"
  email  = "me@example.com"

  letsencrypt_staging = true

  container = {
    image   = "nginxdemos/hello"
  }
}
```

## Example 2: Deploying a `docker-compose.yaml` file

Deploying a Docker Compose file (`docker-compose.yaml`) can provide greater flexibility with regards to how your containers are deployed.

```hcl
module "container-server" {
  source  = "christippett/container-server/cloudinit"
  version = "~> 1.2"

  domain = "example.com"
  email  = "me@example.com"

  files = [
    {
      filename = "docker-compose.yaml"
      content  = filebase64("docker-compose.yaml")
    }
  ]
}
```

Keep in mind when providing your own `docker-compose.yaml` file that you'll need to manually define the labels on your service so that Traefik can identify and route requests to your application.

Traefik is a wonderful tool with a lot of functionality and configuration options, however it can be a bit intimidating to set up if you're not familiar with it. Inspecting the template included in this module is a good starting point if you need help creating your own Docker Compose file. These labels need to be added to every service defined in your Docker Compose file that you want to make available externally.

For more advanced options, refer to the official [Traefik documentation](https://docs.traefik.io/).

```yaml
# docker-compose.yaml

version: "3"

services:
  portainer:
    restart: unless-stopped
    image: portainer/portainer:latest
    command: --admin-password ${PORTAINER_PASSWORD}
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.portainer.rule=Host(`${domain}`)"
      - "traefik.http.routers.portainer.entrypoints=websecure"
      - "traefik.http.routers.portainer.tls=true"
      - "traefik.http.routers.portainer.tls.certresolver=letsencrypt"
networks:
  default:
    external:
      name: web
```

### Some extra information about Traefik

- 🔗 Traefik connects to services over the `web` Docker network by default — all service(s) you want exposed need to be on this network.
- 🔒 Let's Encrypt is configured to use the `letsencrypt` certificate resolver from Traefik. Refer to the example `docker-compose.yaml` file above for the labels needed to enable and configure this feature.
- 📋 Almost all configuration options end up as environment variables defined in a `.env` file saved to the virtual machine. These values are read by Docker Compose on start-up and can be used to parameterise your Docker Compose file without impacting its use in other environments (such as running `docker-compose` locally).
- 📊 The module provides an option for enabling Traefik's [monitoring dashboard](https://docs.traefik.io/operations/dashboard/) and API. When enabled, the dashboard is accessible from `https://${domain}:9000/dashboard/` and the API from `https://${domain}:9000/api/`. The port used by Traefik can be customised using the `TRAEFIK_ADMIN_PORT` environment variable.

# Example integrations with AWS, Google Cloud, Azure and DigitalOcean

The examples below demonstrate creating and deploying virtual machines from different cloud vendors using the cloud-init configuration output from this module.

## AWS

```hcl
resource "aws_instance" "vm" {
  ami             = "ami-0560993025898e8e8" # Amazon Linux 2
  instance_type   = "t2.micro"
  security_groups = ["sg-allow-everything-from-anywhere"]

  tags = {
    Name = "container-server"
  }

  user_data = module.container-server.cloud_config # 👈
}

```

## Google Cloud

```hcl
resource "google_compute_instance" "vm" {
  name         = "container-server"
  project      = "my-project"
  zone         = "australia-southeast1"
  machine_type = "e2-small"
  tags         = ["http-server", "https-server"]

  metadata = {
    user-data = module.container-server.cloud_config # 👈
  }

  boot_disk {
    initialize_params {
      image = data.google_compute_image.cos.self_link
    }
  }

  network_interface {
    subnetwork         = "vpc"
    subnetwork_project = "my-project"

    access_config { }
  }
}

```

## Azure

```hcl
resource "azurerm_linux_virtual_machine" "vm" {
  name                = "container-server"
  resource_group_name = azurerm_resource_group.example.name
  location            = azurerm_resource_group.example.location
  size                = "Standard_F2"
  admin_username      = "adminuser"

  custom_data = base64encode(module.container-server.cloud_config) # 👈

  network_interface_ids = [
    azurerm_network_interface.example.id,
  ]

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Standard_LRS"
  }

  source_image_reference {
    publisher = "Canonical"
    offer     = "UbuntuServer"
    sku       = "20.04-LTS"
    version   = "latest"
  }
}

```

## DigitalOcean

```hcl
resource "digitalocean_droplet" "vm" {
  name   = "container-server"
  image  = "docker-18-04"
  region = "lon1"
  size   = "s-1vcpu-1gb"

  user_data = module.container-server.cloud_config # 👈
}
```

# Terraform Documentation

## Inputs

| Name                | Description                                                                                                                    | Type                                                        | Default | Required |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------- | ------- | :------: |
| domain              | The domain to deploy applications under.                                                                                       | `string`                                                    | n/a     |   yes    |
| email               | The email address used for requesting certificates from Lets Encrypt.                                                          | `string`                                                    | n/a     |   yes    |
| cloudinit_part      | Supplementary cloud-init config used to customise the instance.                                                                | `list(object({ content_type : string, content : string }))` | `[]`    |    no    |
| container           | The container definition used to deploy a Docker image to the server. Follows the same schema as a Docker Compose service.     | `any`                                                       | `{}`    |    no    |
| enable_webhook      | Flag whether to enable the webhook endpoint on the server, allowing updates to be made independent of Terraform.               | `bool`                                                      | `false` |    no    |
| env                 | A list environment variables provided as key/value pairs. These can be used to interpolate values within Docker Compsoe files. | `map(string)`                                               | `{}`    |    no    |
| files               | A list of files to upload to the server. Content must be base64 encoded. Files are available under the `/run/app/` directory.  | `list(object({ filename : string, content : string }))`     | `[]`    |    no    |
| letsencrypt_staging | Boolean flag to decide whether the Let's Encrypt staging server should be used.                                                | `bool`                                                      | `false` |    no    |

## Outputs

| Name                  | Description                                                      |
| --------------------- | ---------------------------------------------------------------- |
| cloud_config          | Content of the cloud-init config to be deployed to a server.     |
| docker_compose_config | Content of the Docker Compose config to be deployed to a server. |
| environment_variables | n/a                                                              |
| included_files        | n/a                                                              |
