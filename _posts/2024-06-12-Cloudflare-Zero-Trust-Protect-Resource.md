# Cloudflare Zero Trust Protect Resource

I started to play around with Cloudflare Zero Trust in order to protect
some resource in AWS.

First: What is Cloudflare Zero Trust? Here a description from Cloudflare:

>Cloudflare Zero Trust provides security without slowdown
>
>Your workforce has expanded to include remote employees, contractors, and vendors. All require secure access to internal applications and tools — no matter where in the world they’re working from.
>
>Cloudflare Zero Trust enables seamless, identity- and context- based application access and software-defined security, allowing you to secure your remote teams, devices, and data without sacrificing performance or user experience.

Source: https://www.cloudflare.com/en-gb/products/zero-trust/remote-workforces/

Requirements:
You need to have a domain in your Cloudflare account in order to be able
to use it and therefore protect resources under that domain.

Let's not wait to long and just go into it :)

We are going to use Terraform code to set up the required resources in Cloudflare.

Create a new directory and open up a new file `provider.tf`

```terraform
terraform {
  required_providers {
    cloudflare = {
      source  = "cloudflare/cloudflare"
      version = "~> 4.33.0"
    }
  }
}

provider "cloudflare" {
  api_token = ""
}

variable "account-id" {
  default = ""
}
```
The Cloudflare API token will require the following permissions:

* `Account: Access: Organizations, Identity Providers, and Groups: Edit`
* `Account: Access: Apps and Policies: Edit`

The account id can be obtained from the Cloudflare Overview dashboard for your
domain.

Next, you create another Terraform file and add the following Terraform resources
to it:

```terraform
variable "whitelist-email-addresses" {
  default = [
    "SomeEmail@domain.whatnot"
  ]
}

resource "cloudflare_access_group" "my-group" {
  account_id = var.account-id
  name       = "my-group"

  include {
    email = var.whitelist-email-addresses
  }
}

resource "cloudflare_access_policy" "my-access-policy-allow" {
  account_id       = var.account-id
  name             = "Allow access"
  decision         = "allow"
  session_duration = "24h"

  include {
    group = [
      cloudflare_access_group.my-group.id
    ]
  }
}

resource "cloudflare_access_application" "my-access-application" {
  account_id       = var.account-id
  name             = "MySecretResource"
  domain           = "domain.to.secret.resource"
  type             = "self_hosted"
  session_duration = "24h"

  policies = [
    cloudflare_access_policy.wireguard-access-policy-allow.id
  ]
}
```

Next you run `terraform init`, `terraform plan`, and if everything is fine
`terraform apply`. This is going to create some Cloudflare resources.

Once applied and resources created, you can visit your application/resource
which is reachable by the provided domain, here `domain.to.secret.resource`.
You should get a window asking you to provide an e-mail address.

Once you type your whitelisted e-mail address, a code is send to you and you
have to type that code into the field. Et voila, you accessed your secret
resource :)