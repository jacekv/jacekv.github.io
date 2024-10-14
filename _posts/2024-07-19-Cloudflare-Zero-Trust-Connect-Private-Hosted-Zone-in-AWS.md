## Connect Cloudflare Zero Trust Tunnel to VPC with private hosted zone in AWS

That's a long title and contains lots of information. Let's break it down:

I have an AWS account with a VPC and a private hosted zone. There are services
running in the private subnets of the VPC that I want to connect to using
the internal DNS names. Now I want to have a tunnel, which allows me to connect
to these services using the private DNS names from my local machine using the
Warp client.

Comprendre? Let's go!

### Prerequisites

You have an AWS account with a VPC and a private hosted zone. You have services
running in the private subnets of the VPC that you want to connect to using the
internal DNS names. You created records in the private hosted zone for these
services.

You should also have a Cloudflare Account :)

If everything is up and ready, let's start!

### Step 1: Create a Cloudflare Zero Trust Tunnel

Go to your Cloudflare Zero Trust dashboard and create a new tunnel. To create
a new tunnel, go to `Networks -> Tunnels` and click on `Add a tunnel`.

For the tunnel type, select `Cloudflared` and click on `Next`.

Give your tunnel a nice name and click on `Save tunnel`.

Next, you should see a window with the configuration details for cloudflared.

I decided to use the docker version, by building my own Docker image and
running it in an ECS Cluster. You can also just setup a EC2 and run it there -
important is that you place it in the private subnet of your VPC.

No need to expose the cloudflared connector to the internet.

Let me show you an example of the Dockerfile I am using:

```Dockerfile
# ==============================================================================
# Download cloudflared
# ==============================================================================
FROM --platform=linux/amd64 debian:bookworm-slim as build

# Install dependencies
RUN apt-get update                       && \
    apt-get install -y curl

# Download cloudflared
RUN curl -L https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64 -o cloudflared  && \
    chmod +x cloudflared                && \
    mv cloudflared /usr/local/bin/

# ==============================================================================
# Run from distroless
# ==============================================================================
FROM alpine:3.12

# Get dependencies
COPY --from=build /usr/local/bin/cloudflared /usr/local/bin/cloudflared

# Copy script
WORKDIR /etc/cloudflared
COPY ./entrypoint.sh /etc/cloudflared/entrypoint.sh
RUN chmod +x /etc/cloudflared/entrypoint.sh

# Run script
ENTRYPOINT ["/etc/cloudflared/entrypoint.sh"]
```

And the entrypoint.sh:

```bash
cloudflared tunnel --no-autoupdate run --token <token>
```
where `<token>` is the token you get from Cloudflare.

And that's it. Deploy it which way you prefer wait until it is up and running.

If you wait for the cloudflared connector to come up, you will see it in the
same window as the token is. You can also go back to the starting tunnels page
and see when the status of the tunnel changes from `INACTIVE` to `HEALTHY`.

### Step 2: Access group

Let's create an access group which will make our life a bit easier on managing
users joining our tunnel and what not.

Go to `Access -> Access Groups` and click on `Add a group`.

Give it a nice name, some criteria which allows users to join the group,
like email or something else, and click on `Save`.

You should see your newly created `Access Group` in the list.

### Step 3: Setting up device enrollment permissions for Warp

Now it is time to configure the device enrollment permissions for the Warp client.

With this configuration you define who is able to join our tunnel.

In the Dashboard, go to `Settings -> Warp Client -> Device enrollment permissions -> Manage`
and add a rule. Instead of creating a new rule alltogether, we are going to
add the newly created `Access Group` to the existing rule.

For this, tick the box in front of the `Access Group` name in the
`Assign a group` section and provide a rule name. Once you are done, click on
`Save`.

### Step 4: Configure Split Tunneling

Split Tunnels can be configured to exclude or include IP addresses or domains
from going through WARP. This feature is commonly used to run WARP alongside a
VPN (in Exclude mode) or to provide access to a specific private
network (in Include mode).

I decided to go with the Exclude mode, since I was not able to make the Include
mode work properly. I will get back to this later, on what hasn't been working
for me.

To configure the split tunneling, go to
`Settings -> Warp Client -> Device settings -> Profile settings -> [select a profile] -> Split Tunnels -> Manage`

Now we have to do a bit of CIDR calculation. Let's assume your VPC has the CIDR
`172.18.0.0/16`. In the list you can see that `172.16.0.0/12` is already
excluded, but that's to big.

Remove the `172.16.0.0/12` and add the following CIDRs:

* `172.24.0.0/13`
* `172.20.0.0/14`
* `172.16.0.0/15`
* `172.19.0.0/16`

This should direct all `172.18.0.0/16` traffic through the tunnel.

### Step 5: Install Warp and login

Next it is time to install the Warp client on your machine and login with your
account, which has been whitelisted in the `Access Group`.

To see if everything is fine, go to `https://help.teams.cloudflare.com/`. You
should see the `Your network is connected` message and the proper team name
on the website.

If yes, you did everything correct :)

Earlier I mentioned that I had some issues with the Include mode. I was not able
to make this help page working with the Include mode, which is why I decided
to go with the Exclude mode.

If someone has an idea on how to make the Include mode work, please let me know.

### Step 6: Add private DNS resolver

We want to reach our internal services with DNS entries from the private hosted
zone in AWS. In order to know which DNS resolver to use, check the
[AWS docs](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver-overview-DSN-queries-to-vpc.html).

It turns out, that the DNS resolver for a VPC has the IP address `VPC + 2`. That means
if your VPC CIDR is `172.18.0.0/16`, the DNS resolver IP address is `172.18.0.2`.

To add the DNS resolver to our Cloudflare Zero Trust Tunnel, go to
`Settings -> WARP Client -> Profile -> [default or something else] -> Local Domain Fallback -> Manage`
and add the DNS resolver IP address.

And that's it :)

### Step 7: Test

Now it is time to test if everything is working as expected. Try to reach whatever
private service you have in your VPC using the internal DNS name.

In case it does not work, check if the security group of the Cloudflared connector
is whitelisted in the ingress rules of the security group of the service you
want to reach.

### Conclusion

You have successfully connected your local machine to your VPC with a private
hosted zone in AWS using Cloudflare Zero Trust Tunnel. You can now access your
internal services using the internal DNS names without exposing those to the internet.

I hope this guide was helpful and you were able to follow along. If you have any
questions or feedback, feel free to reach out to me.

Happy tunneling! 🚀

Maybe next time I will have some Terraform code for you to automate the setup.