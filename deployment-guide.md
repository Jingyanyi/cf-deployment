# Deploying CF

## Pave your IaaS and get a BOSH Director

Before you can start deploying,
you'll need to make sure you've configured your infrastructure appropriately
and deployed a BOSH Director.
If you're using AWS, GCP, or Azure,
we'd suggest using [bbl](https://github.com/cloudfoundry/bosh-bootloader)
to set up your IaaS resources and bootstrap a BOSH director.
Otherwise, take a look at [the BOSH documentation](https://bosh.io/docs/init.html)
for information about prerequisites for a given IaaS
and installing a BOSH Director there.

#### bosh-lite
If you're deploying bosh-lite to a VM on AWS, GCP, or Azure,
look at [this guide](bosh-lite.md).

If you're planning to deploy against a **local** bosh-lite,
follow [these instructions](https://bosh.io/docs/bosh-lite.html).
You'll also need to take the following step before continuing:
```
export BOSH_CA_CERT=<PATH-TO-BOSH-LITE-REPO>/ca/certs/ca.crt
```

### Get load balancers
The CF Routers need a way to receive traffic.
The most common way to accomplish this is to configure load balancers
to route traffic to them.
While we cannot offer help for each IaaS specifically,
for IaaSes like AWS and GCP,
you can use `bbl` to create load balancers
by running `bbl create-lbs --type cf --domain <SYSTEM_DOMAIN>`.
(`bbl` support for creating load balancer on Azure is coming soon.)

#### On certificates
Before you can create your load balancers,
you'll need to be able to provide an SSL certificate
for the domain that your load balancers will use.
You might already have one,
especially if you've already used this domain for a previous environment.

If you're deploying a fresh environment with a new domain,
you can generate a self-signed cert.
**Don't forget that the common name should match your intended system domain (and app domains, if it's different)**:
```
openssl req -x509 -newkey rsa:2048 -keyout key.pem -out cert.pem -nodes
```

Alternatively, if you want to get a certificate from a trusted Certificate Authority,
you can skip this cert generation step
and provide that certificate directly to the `bbl` command below.

### Update your DNS records to point to your load balancer
Once you have a load balancer set up,
you'll want to make sure that your system and app domains resolve to the appropriate load balancer.

You can set up DNS with your preferred provider,
but if you've used `bbl` to create your load balancers on GCP or AWS
(support for Azure is on the way),
`bbl` will create NS records for your system domain.
If you manage your DNS with some other provider
-- for example, with Route53 --
you can copy the NS record data that `bbl` created,
and paste it into the `value` section of the Route53 NS record for your domain.
 
After a few minutes,
the your system domain should resolve to your load balancer.

### (For `bbl` users) Save `bbl-state.json`
However you run `bbl` (command line or with Concourse),
the side-effect of a successful bbl command is the creation/update of `bbl-state.json`.
As a deployer, **you must persist this file somehow.**

Currently, our Concourse tasks assume
that you want to check this file into a private git repo.
We'll likely prioritize work soon
to persist that file to a more secure location such as Lastpass.

## Target your BOSH Director
There are several ways to target your new BOSH director.
One of the simplest ways is to create an environment alias:
```
bosh -e <DIRECTOR_IP> alias-env my-env --ca-cert DIRECTOR_CA_CERT_FILE

bosh -e my-env login
# Provide username and password for your BOSH Director
```

Alternatively, you can set environment variables:
```
export BOSH_ENVIRONMENT=<DIRECTOR_IP>
export BOSH_CLIENT=<DIRECTOR_USERNAME>
export BOSH_CLIENT_SECRET=<DIRECTOR_PASSWORD>
export BOSH_CA_CERT=<DIRECTOR_CA_CERT_TEXT>
```

If you've used `bbl` to set up your director,
you can fetch the director location and credentials with the following commands:

```
eval "$(bbl print-env)"
```
or
```
export BOSH_ENVIRONMENT=$(bbl director-address)
export BOSH_CLIENT=$(bbl director-username)
export BOSH_CLIENT_SECRET=$(bbl director-password)
export BOSH_CA_CERT="$(bbl director-ca-cert)"
```

## Upload a `cloud-config`
cf-deployment depends on use of a [cloud-config](https://bosh.io/docs/cloud-config).
You can find details about the requirements for a cloud-config [here](texts/on-cloud-configs.md),
but if you used `bbl` to set up your BOSH director,
`bbl` already uploaded a valid cloud-config for you.

For bosh-lite,
```
bosh -e MY_ENV update-cloud-config bosh-lite/cloud-config.yml
```

## Upload a stemcell

Upload the current stemcell for `cf`
by running the command below with the appropriate
stemcell type and version number.
The stemcell type will vary by IaaS provider
(you can find a list of stemcells at http://bosh.io/stemcells),
and the version number is specified on the last line of `cf-deployment.yml`.
```
bosh upload-stemcell https://bosh.io/d/stemcells/bosh-IAAS_INFO-ubuntu-trusty-go_agent?v=VERSION
```

## Deploy CF
To deploy to a configured BOSH director using the new `bosh` CLI:

```
export SYSTEM_DOMAIN=some-domain.that.you.have
bosh -e my-env -d cf deploy cf-deployment/cf-deployment.yml \
  --vars-store env-repo/deployment-vars.yml \
  -v system_domain=$SYSTEM_DOMAIN \
  [ -o operations/CUSTOMIZATION1 ] \
  [ -o operations/CUSTOMIZATION2 (etc.) ]
```

The CF Admin credentials will be stored in the file passed to the `--vars-store` flag
(`env-repo/deployment.yml` in the example).
You can find them by searching for `cf_admin_password`.

If you're using a local bosh-lite,
remember to add the `operations/bosh-lite.yml` ops-file
to your deploy command:
  ```

  bosh -e 192.168.50.6 -d cf deploy cf-deployment.yml \
    -o operations/bosh-lite.yml \
    --vars-store deployment-vars.yml \
    -v system_domain=bosh-lite.com
  ```
