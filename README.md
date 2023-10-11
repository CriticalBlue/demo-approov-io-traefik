# AWS EC2 TRAEFIK SETUP FOR DEMO.APPROOV.IO

[Traefik](https://traefik.io/traefik/) setup to run all docker containers on an AWS EC2 instance behind the same ports, 80 and 443, with automated Let'sEncrypt certificates creation and renewal.


## CREATE AN AWS EC2 INSTANCE

Create a new AWS EC2 instance, otherwise you need to ensure that the existing one you are re-using does not have anything listening on port `80` or port `443`.
The instance needs to have about 4GB RAM and 16GB disk space for building the docker containers for the services. Use a t3.medium initially - this can be reduced to a t3.small after setup is complete.
To resize an instance, stop the instance, select it and then
- To change the type: Actions → Instance Settings → Change Instance Type (see Amazon EC2 T3 Instances)
- To change the amount of storage: in the instance details’ Storage tab, click on the volume ID, then Actions → Modify volume.

You will need to login to the instance, so you need to add an SSH key pair to the EC2 instance (see [Add or remove a public key on your instance - AWS](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/replacing-key-pair.html).

Make sure inbound rules allow access to the instance. Enable SSH access on port 22 from your current IP in the instance's security policy.
For example (security group rule IDs and source for SSH will differ based on your configuration and location):

| Security group rule ID | IP version | Type       | Protocol | Port range | Source     |
| ---------------------- | ---------- | ---------- | -------- | ---------- | ---------- |
| sgr-0123456789abcdef0  | IPv4       | HTTP       | TCP      | 80         | 0.0.0.0/0  |
| sgr-0123456789abcdef0  | IPv4       | HTTPS      | TCP      | 443        | 0.0.0.0/0  |
| sgr-0123456789abcdef0  | IPv4       | SSH        | TCP      | 22         | `<your IP address>`/32 |

Attach an elastic IP to the instance to permit starting and stopping of the instance without having to update the IP in DNS. Take note of the elastic IP address of the EC2 instance for use in the next step.


## SET UP DNS DOMAINS

Before starting the setup of services, a domain needs to be configured for the EC2 instance. These instructions are for the domains `demo.approov.io` and `shapes.approov.io` in AWS Route 53 (but would work similarly for any other provider). Each service added to the demo server will use one of these as its base domain. So when adding a backend for the python shapes API you could give it the domain `python-shapes.demo.approov.io`, and for nodejs `nodejs-shapes.demo.approov.io`, respectively.

Configure a domain at Route 53 and point it to the elastic IP address from the previous step. Create a new hosted zone (AWS Console → Route 53 → Hosted Zones → Create hosted zone) called `demo.approov.io`. Ensure that an A-type DNS record pointing to the EC2 instance's elastic IP exists in the hosted zone `demo.approov.io`. Also add an A-type wild-card DNS record (`*.demo.approov.io`) to route any sub-domain to the same IP. Replicate the auto generated NS-type record in the hosted zone `demo.approov.io` to the hosted zone approov.io in order to make the new hosted zone’s name servers known in `approov.io`. Repeat these steps for `shapes.approov.io`.


## SET UP THE AWS EC2 INSTANCE

Log into the demo server AWS EC2 instance using the ssh private key from the key pair that you used when you created the EC2 instance. The destination for login is `ec2-user@demo.approov.io`:
```bash
ssh -i /path/key-pair-name.pem ec2-user@demo.approov.io
```

A more convenient way is to use key management software, for example saving the key in KeePassXC and using this to add the key to SSH agent when needed. This allows login without providing the key on the command line:
```bash
ssh ec2-user@demo.approov.io
```

### Install Git

Once logged in to the EC2 instance, install Git.

```
yum install -y --skip-broken git
```

### Traefik Setup

#### Cloning this repository

Start by cloning this repository:

```
git clone https://github.com/CriticalBlue/demo-approov-io-traefik.git && cd demo-approov-io-traefik
```

#### Run the setup

Traefik, Docker and Docker Compose will be installed and configured by running the bash script in the root of this repo:

```
./aws-ec2-setup.sh
```

The end of the output will look like this:

```
---> DOCKER COMPOSE VERSION <---
docker-compose version 1.25.5, build 8a1c60f6


---> GIT VERSION<---
git version 2.23.3


---> TRAEFIK installed at: /opt/traefik <---

From /opt/traefik folder you can ran any docker-compose command.

Some useful examples:

## Restart Traefik:
sudo docker-compose restart traefik

## Start Traefik:
sudo docker-compose up -d traefik

## Destroy Traefik:
sudo docker-compose down

## Tailing the Traefik logs in realtime:
sudo docker-compose logs --follow traefik

---> TRAEFIK is now listening for new docker containers <---
```

This setup script will let Traefik running and listening for incoming requests on port `80` and `443`, where requests for port `80` will be redirected to port `443`.


## TLS CERTIFICATES

Traefik uses LetsEncrypt to automatically generate and renew TLS certificates for all domains it is listening on, and it will keep the public key unchanged, thus a mobile app can implement certificate pinning against the public key without the concern of having the pin changed at each renewal of the certificate. Let's Encrypt issues wildcard certificates only through using a DNS-01 challenge.

### SET UP ROUTE 53 FOR A DNS-01 CHALLENGE

Give the demo server EC2 instance an IAM role that permits it to write the necessary TXT-type record for the DNS-01 challenge to Route 53 DNS.
1. Create an IAM policy (AWS → IAM → Policies → Create policy) called demo-approov-io-dns-01, with description “Minimum permissions required for a DNS-01 challenge performed by Traefik”, the hosted zone IDs of `demo.approov.io` and `shapes.approov.io` and the correct TXT-type record names derived from the corresponding domain names:
```json
  {
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": "route53:GetChange",
        "Resource": "arn:aws:route53:::change/*"
      },
      {
        "Effect": "Allow",
        "Action": "route53:ListHostedZonesByName",
        "Resource": "*"
      },
      {
        "Effect": "Allow",
        "Action": [
          "route53:ListResourceRecordSets"
        ],
        "Resource": [
          "arn:aws:route53:::hostedzone/<hosted zone ID of demo.approov.io>"
        ]
      },
      {
        "Effect": "Allow",
        "Action": [
          "route53:ListResourceRecordSets"
        ],
        "Resource": [
          "arn:aws:route53:::hostedzone/<hosted zone ID of shapes.approov.io>"
        ]
      },
      {
        "Effect": "Allow",
        "Action": [
          "route53:ChangeResourceRecordSets"
        ],
        "Resource": [
          "arn:aws:route53:::hostedzone/<hosted zone ID of demo.approov.io>"
        ],
        "Condition": {
          "ForAllValues:StringEquals": {
          "route53:ChangeResourceRecordSetsNormalizedRecordNames": [
              "_acme-challenge.demo.approov.io"
            ],
            "route53:ChangeResourceRecordSetsRecordTypes": [
              "TXT"
            ]
          }
        }
      },
      {
        "Effect": "Allow",
        "Action": [
          "route53:ChangeResourceRecordSets"
        ],
        "Resource": [
          "arn:aws:route53:::hostedzone/<hosted zone ID of shapes.approov.io>"
        ],
        "Condition": {
          "ForAllValues:StringEquals": {
          "route53:ChangeResourceRecordSetsNormalizedRecordNames": [
              "_acme-challenge.shapes.approov.io"
            ],
            "route53:ChangeResourceRecordSetsRecordTypes": [
              "TXT"
            ]
          }
        }
      }
    ]
  }
```
2. Create an IAM role (IAM → Roles → Create role) with trusted entity type "AWS service" and use case "EC2". Next, select the policy demo-approov-io-dns-01 and enter the role name "demo-server" and description "Allows demo-server EC2 instance to perform a DNS-01 challenge for `demo.approov.io` and shapes.approov.io through Traefik and LetsEncrypt".
3. Add the IAM role to the demo server EC2 instance (AWS → EC2 → Instances. Select the instance Combined Demo Server. Actions → Security → Modify IAM role. Select IAM Role: demo-server, Update IAM role).
4. Restart Traefik to allow it to request the certificate from LetsEncrypt.

This completes the setup. Read on for examples on how to configure and deploy a service to run on the server.


## EXAMPLE: DEPLOYING A SERVICE

Let's see an example of deploying the Python Shapes API backend into an EC2 instance listening at `*.demo.approov.io`.

#### Clone the Repo

```
git clone https://github.com/approov/python-flask_approov-shapes-api-server && cd python-flask_approov-shapes-api-server
```

#### Create the .env file

```
cp .env.example .env
```

#### Edit the .env file

Replace the default domain with your own server domain:

```bash
PYTHON_FLASK_SHAPES_DOMAIN=python-shapes.demo.approov.io
```

Replace the dummy Approov secret on it with the one for your Approov account:

```bash
# approov secret -get base64
APPROOV_BASE64_SECRET=your-secret-here
```

#### Start the Docker Stack

```
sudo docker-compose up -d
```

Now visit `python-shapes.demo.approov.io` in your browser to check the server is accepting requests.

#### Inspect the logs

```
sudo docker-compose logs -f
```

## EXAMPLE: ADDING A CONTAINER TO TRAEFIK

> **NOTE:** The following is not required for the above Deploying a Service Example. You only need to follow this part if your project does not have Traefik labels in the `docker-compose.yml` file.

Traefik inspects the labels in all running docker containers to know for which ones it needs to proxy requests.

So if your backend does not yet have support for Traefik in the `docker-compose.yml` file, you can configure your service like this:

```yml
services:

    api:
        ...

        labels:
            - "traefik.enable=true"

            # The public domain name for your docker container
            - "traefik.frontend.rule=Host:api.demo.approov.io"

            # Does not need to be exactly the same as the domain name.
            - "traefik.backend=api.demo.approov.io"

            # The external docker network that Traefik uses to proxy request to containers.
            - "traefik.docker.network=traefik"

            # This is the internal container port, not the public one.
            - "traefik.port=5000"
...

networks:
    ...

    traefik:
        external: true

```

With this configuration all requests for `https://api.demo.approov.io` will be proxy by Traefik to the docker container with the backend label `traefik.backend=api.demo.approov.io` on the internal container network port `traefik.port=5000`.
