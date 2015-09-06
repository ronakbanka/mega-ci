#Deploying Concourse

## Requirements

* An AWS account for your Concourse deployment. It doesn't need to be empty as
  we can contain everything inside a VPC.

* The `aws` command line tool. This can be installed by running `brew install awscli`
  if you have Homebrew installed. You should run `aws configure` after
  installation to authenticate the CLI.

* The `bosh` command line tool.  This can be installed by running `gem install bosh_cli`

* The `bosh-init` command line tool. Instructions for installation can be found
  [here][bosh-init-docs].

* The `jq` command line tool. This can be installed by running `brew install jq`
  if you have Homebrew installed.

* The `spiff` command line tool. The latest release can be found [here]
  [spiff-releases].

* A deployment directory with the following minimal skeleton structure:
```
my_deployment_dir/
|- aws_environment
|- certs/
|  |- concourse.pem
|  |- concourse.key
|  |- (concourse_chain.pem, optional)
|- cloud_formation/
|  |- (properties.json, optional)
|- stubs/
   |- bosh/
   |  |- bosh_passwords.yml
   |- concourse/
      |- atc_credentials.yml
      |- binary_urls.json

```

### Deployment Directory Details

The `aws_environment` file should look like this:

```bash
export AWS_DEFAULT_REGION=REPLACE_ME # e.g. us-east-1
export AWS_ACCESS_KEY_ID=REPLACE_ME
export AWS_SECRET_ACCESS_KEY=REPLACE_ME
```

You need an SSL certificate for the domain where Concourse will be accessible. The
key and pem file must exist at `certs/concourse.key` and `certs/concourse.pem`. If
there is a certificate chain, it should exist at `certs/concourse_chain.pem`.
You can generate a self signed cert if needed:
                                                                             
* `openssl genrsa -out concourse.key 1024`
* `openssl req -new -key concourse.key -out concourse.csr` For the Common Name, you must enter your self signed domain.
* `openssl x509 -req -in concourse.csr -signkey concourse.key -out concourse.pem`
* Copy `concourse.pem` and `concourse.key` into the certs directory.

The optional `cloud_formation/properties.json` file should look like this:

```json
[
  {
    "ParameterKey": "ConcourseHostedZoneName",
    "ParameterValue": "REPLACE_WITH_HOSTED_ZONE_NAME"
  },
  {
    "ParameterKey": "ELBRecordSetName",
    "ParameterValue": "REPLACE_WITH_HOST_NAME"
  }
]

```
If both `ConcourseHostedZoneName` and `ELBRecordSetName` are provided, a Route 53 hosted zone will be created with the given 
`ConcourseHostedZoneName` name, and a DNS entry pointing at the new ELB will be created with the given
`ELBRecordSetName` name.


The `stubs/bosh/bosh_passwords.yml` should look like this:

```yaml
bosh_credentials:
  agent_password: REPLACE_WITH_PASSWORD
  director_password: REPLACE_WITH_PASSWORD
  mbus_password: REPLACE_WITH_PASSWORD
  nats_password: REPLACE_WITH_PASSWORD
  redis_password: REPLACE_WITH_PASSWORD
  postgres_password: REPLACE_WITH_PASSWORD
  registry_password: REPLACE_WITH_PASSWORD
```

The `stubs/concourse/atc_credentials.yml` file should look like this:

```yaml
atc_credentials:
  basic_auth_username: REPLACE_ME
  basic_auth_password: REPLACE_ME
  db_name: REPLACE_ME
  db_user: REPLACE_ME
  db_password: REPLACE_ME
```

Finally, the `stubs/concourse/binary_urls.json` should look something like this:

```json
{
  "stemcell": "https://d26ekeud912fhb.cloudfront.net/bosh-stemcell/aws/light-bosh-stemcell-3068-aws-xen-hvm-ubuntu-trusty-go_agent.tgz",
  "concourse_release": "https://bosh.io/d/github.com/concourse/concourse?v=0.62.0",
  "garden_release": "https://bosh.io/d/github.com/cloudfoundry-incubator/garden-linux-release?v=0.303.0"
}
```

You can find the latest stemcells [here][bosh-stemcells], and the latest
concourse (and associated garden releases) [here][concourse-releases].

## Setting up Your AWS Environment and Deploying BOSH

Run:

```bash
./scripts/deploy_bosh PATH_TO_DEPLOYMENT_DIR
```

This will execute the AWS cloud formation template and then create a BOSH
instance. The script generates several artifacts in your deployment directory:

* `artifacts/deployments/bosh.yml`: the deployment manifest for your BOSH instance
* `artifacts/deployments/bosh-state.json`: an implementation detail of `bosh-init`;
  used to determine things like whether it is deploying a new BOSH or updating an
  existing one
* `artifacts/keypair/id_rsa_bosh`: the private key created in your AWS
  account that will be used for all deployments; you will need this if you ever
  want to ssh into the BOSH instance or any of the concourse instances.

The script will also print the IP of the BOSH director. Target your bosh by running:
```bash
bosh target DIRECTOR_IP
```

The default username/password is admin/admin. You are **strongly advised** to change
these by running:

```bash
bosh create user USERNAME PASSWORD
```

When you're done, create a file called `bosh_environment` at the root of your
deployment directory that looks like this:

```bash
export BOSH_USER=REPLACE_ME
export BOSH_PASSWORD=REPLACE_ME
export BOSH_DIRECTOR=https://REPLACE_ME_WITH_BOSH_DIRECTOR_IP:25555
```

## Deploying Concourse

Run:

```bash
./scripts/deploy_concourse PATH_TO_DEPLOYMENT_DIR
```

The script will deploy Concourse. It generates one additional artifact in your
deployment directory:

* `artifacts/deployments/concourse.yml`: the deployment manifest of your Concourse

The script will also print the Concourse load balancer hostname at the end. This can be
used to create the `CNAME` for your DNS entry in Route53 so that you can have a nice
URL where you access your Concourse.

#Deploying Cloud Foundry

## Requirements

* An AWS account for your Cloud Foundry deployment. It doesn't need to be empty as
  we can contain everything inside a VPC.

* The `aws` command line tool. This can be installed by running `brew install awscli`
  if you have Homebrew installed. You should run `aws configure` after
  installation to authenticate the CLI.

* The `bosh` command line tool.  This can be installed by running `gem install bosh_cli`

* The `bosh-init` command line tool. Instructions for installation can be found
  [here][bosh-init-docs].

* The `jq` command line tool. This can be installed by running `brew install jq`
  if you have Homebrew installed.

* The `spiff` command line tool. The latest release can be found [here]
  [spiff-releases].

* A deployment directory with the following minimal skeleton structure:
```
my_deployment_dir/
|- aws_environment
|- certs/
|  |- cf.pem
|  |- cf.key
|- cloud_formation/
|  |- buckets-properties.json
|  |- cf-properties.json
|- stubs/
   |- bosh/
   |  |- bosh_passwords.yml
   |- cf/
      |- cf-stub.yml

```

### Deployment Directory Details

The `aws_environment` file should look like this:

```bash
export AWS_DEFAULT_REGION=REPLACE_ME # e.g. us-east-1
export AWS_ACCESS_KEY_ID=REPLACE_ME
export AWS_SECRET_ACCESS_KEY=REPLACE_ME
```

You need an SSL certificate for the domain where Cloud Foundry will be accessible. The key and pem file must 
exist at `certs/cf.key` and `certs/cf.pem`. You can generate a self signed cert if needed:

* `openssl genrsa -out cf.key 1024`
* `openssl req -new -key cf.key -out cf.csr` For the Common Name, you must enter "*." followed by your self signed domain.
* `openssl x509 -req -in cf.csr -signkey cf.key -out cf.pem`
* Copy `cf.pem` and `cf.key` into the certs directory.

The `cloud_formation/buckets-properties.json` file should look like this:

```json
[
  {
    "ParameterKey": "CCBuildpacksBucketName",
    "ParameterValue": "REPLACE_WITH_YOUR_SYSTEM_DOMAIN-cc-buildpacks"
  },
  {
    "ParameterKey": "CCDropletsBucketName",
    "ParameterValue": "REPLACE_WITH_YOUR_SYSTEM_DOMAIN-cc-droplets"
  },
  {
    "ParameterKey": "CCPackagesBucketName",
    "ParameterValue": "REPLACE_WITH_YOUR_SYSTEM_DOMAIN-cc-packages"
  },
  {
    "ParameterKey": "CCResourcesBucketName",
    "ParameterValue": "REPLACE_WITH_YOUR_SYSTEM_DOMAIN-cc-resources"
  },
  {
    "ParameterKey": "CloudFrontOriginAccessIdentityId",
    "ParameterValue": "OPTIONAL-REPLACE_WITH_CLOUD_FRONT_ORIGIN_ACCESS_ID_IF_USING_CLOUD_FRONT_FOR_BLOBSTORE_CDN"
  },
  {
    "ParameterKey": "AwsAccountId",
    "ParameterValue": "OPTIONAL-REPLACE_WITH_AWS_ACCOUNT_ID_IF_USING_CLOUD_FRONT_FOR_BLOBSTORE_CDN"
  },
  {
    "ParameterKey": "AcceptanceTestLogsBucketName",
    "ParameterValue": "OPTIONAL-REPLACE_WITH_BUCKET_NAME_IF_YOU_WANT_A_BUCKET_FOR_STORING_CATS_LOGS"
  }
]

```

#### Optional Instructions for using Cloud Front

To configure Cloud Front as a CDN for your Resource Matching and Droplet blobstores:
 
1. Navigate to the Cloud Front configuration page in the AWS Console.
2. Click `Origin Access Identity` in the left column.
3. Click `Create Origin Access Identity`
4. Click `Create` in the modal window.
5. Copy the `ID` for the new identity, and use it as the `CloudFrontOriginAccessIdentityId` in `buckets-properties.json`
6. At the top of the AWS Console, click your account name, then select `My Account` from the drop down.
7. Copy the `Account Id` and use it as the `AwsAccountId` in `buckets-properties.json`

The `cloud_formation/cf-properties.json` file should look like this:

```json
[
  {
    "ParameterKey": "CFHostedZoneName",
    "ParameterValue": "REPLACE_WITH_SYSTEM_DOMAIN_NAME_OF_THE_CLOUD_FOUNDRY_INSTALLATION"
  },
  {
    "ParameterKey": "CFAppsDomainHostedZoneName",
    "ParameterValue": "REPLACE_WITH_THE_APPS_DOMAIN_NAME__CAN_BE_IDENTICAL_TO_THE_SYSTEM_DOMAIN_NAME"
  },
  {
    "ParameterKey": "CCDBUsername",
    "ParameterValue": "CHOOSE_A_USERNAME_FOR_THE_CCDB_RDS_DATABASE"
  },
  {
    "ParameterKey": "CCDBPassword",
    "ParameterValue": "CHOOSE_A_PASSWORD_FOR_THE_CCDB_RDS_DATABASE"
  },
  {
    "ParameterKey": "UAADBUsername",
    "ParameterValue": "CHOOSE_A_USERNAME_FOR_THE_UAADB_RDS_DATABASE"
  },
  {
    "ParameterKey": "UAADBPassword",
    "ParameterValue": "CHOOSE_A_PASSWORD_FOR_THE_UAADB_RDS_DATABASE"
  }
]
```

The `stubs/bosh/bosh_passwords.yml` should look like this:

```yaml
bosh_credentials:
  agent_password: REPLACE_WITH_PASSWORD
  director_password: REPLACE_WITH_PASSWORD
  mbus_password: REPLACE_WITH_PASSWORD
  nats_password: REPLACE_WITH_PASSWORD
  redis_password: REPLACE_WITH_PASSWORD
  postgres_password: REPLACE_WITH_PASSWORD
  registry_password: REPLACE_WITH_PASSWORD
```

The `stubs/cf/cf-stub.yml` will be your primary manifest stub for your Cloud Foundry deployment. Any changes you
want to make should be made here. Certain values will be merged into this stub by default. See the `example_stubs/cf`
directory for example stubs. Also read `example_stubs/cf/README` for instructions on filling out the stubs.

## Setting up Your AWS Environment and Deploying BOSH

Run:

```bash
./scripts/deploy_cf_bosh_instance PATH_TO_DEPLOYMENT_DIR
```

This will execute the AWS cloud formation template and then create a BOSH
instance. The script generates several artifacts in your deployment directory:

* `artifacts/deployments/cf_bosh.yml`: the deployment manifest for your BOSH instance
* `artifacts/deployments/cf_bosh-state.json`: an implementation detail of `bosh-init`;
  used to determine things like whether it is deploying a new BOSH or updating an
  existing one
* `artifacts/keypair/id_rsa_bosh`: the private key created in your AWS
  account that will be used for all deployments; you will need this if you ever
  want to ssh into the BOSH instance or any of the concourse instances.
* `generated-stubs/cf/*`: a series of files with pieces of your cf deployment manifest
  that are generated from Cloud Formation data.

The script will also print the IP of the BOSH director. Target your bosh by running:
```bash
bosh target DIRECTOR_IP
```

The default username/password is admin/admin. You are **strongly advised** to change
these by running:

```bash
bosh create user USERNAME PASSWORD
```

When you're done, create a file called `bosh_environment` at the root of your
deployment directory that looks like this:

```bash
export BOSH_USER=REPLACE_ME
export BOSH_PASSWORD=REPLACE_ME
export BOSH_DIRECTOR=https://REPLACE_ME_WITH_BOSH_DIRECTOR_IP:25555
```


## Deploying Cloud Foundry

You can deploy Cloud Foundry using either [cf-deployment][cf-deployment] or [cf-release][cf-release]. In either case,
when you reference stub files, or generate the deployment manifest, use 
`DEPLOYMENT_DIRECTORY/stubs/cf/cf-stub.yml DEPLOYMENT_DIRECTORY/generated-stubs/cf/*`.


[concourse-releases]: https://github.com/concourse/concourse/releases
[bosh-init-docs]: https://bosh.io/docs/install-bosh-init.html
[bosh-stemcells]: http://bosh.io/stemcells
[spiff-releases]: https://github.com/cloudfoundry-incubator/spiff/releases
[cf-deployment]: https://github.com/cloudfoundry/cf-deployment
[cf-release]: https://github.com/cloudfoundry/cf-release
