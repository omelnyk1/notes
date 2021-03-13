# How to restore deleted object in GCS bucket

## Problem
We accidently deleted Terraform state for test project (test.tfstate) in bucket tf-state. And now we need to obtain some resource data from latest available object version (generation).

## Solution
- Make sure versioning feature is enabled for state bucket:
```sh
$ gsutil versioning get gs://tf-state/
gs://tf-state: Enabled
```

- Check live generation for object test.tfstate:
```sh
$ gsutil ls gs://tf-state/test.tfstate
CommandException: One or more URLs matched no objects.
```

- List available non-current generations for object test.tfstate:
```sh
$ gsutil ls -la gs://tf-state/test.tfstate
       157  2020-05-25T06:28:57Z  gs://tf-state/test.tfstate#1590388137129660  metageneration=1
     91576  2020-05-25T08:25:18Z  gs://tf-state/test.tfstate#1590395118548302  metageneration=1
    123117  2020-05-25T08:34:40Z  gs://tf-state/test.tfstate#1590395680684896  metageneration=1
    ...
    394424  2020-08-04T16:10:10Z  gs://tf-state/test.tfstate#1596557410470854  metageneration=1
```

- Copy the latest available generation (1596557410470854):
```sh
$ gsutil cp gs://tf-state/test.tfstate#1596557410470854 .
Copying gs://tf-state/test.tfstate#1596557410470854...
/ [1 files][385.2 KiB/385.2 KiB]
Operation completed over 1 objects/385.2 KiB.
```

- Open resored state file:
```sh
$ head -18 test.tfstate
{
  "version": 4,
  "terraform_version": "0.12.26",
  "serial": 185,
  "lineage": "23276b6c-365f-5b40-8769-12f73181c945",
  "outputs": {},
  "resources": [
    {
      "module": "module.waf",
      "mode": "data",
      "type": "cloudflare_ip_ranges",
      "name": "main",
      "provider": "module.waf.provider.cloudflare",
      "instances": [
        {
          "schema_version": 0,
          "attributes": {
            "cidr_blocks": [
...

```
> :warning:
Command gsutil rm -a removes all object versions (generations and metageneration). In that case we won’t restore objects.
