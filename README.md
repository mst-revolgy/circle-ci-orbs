# Circle CI Helpers

## publishing to Circle CI

1. claim the namespace

```bash
circleci namespace create purple-technology github purple-technology
```

2. create the orb

```bash
circleci orb create purple-technology/eks-helpers
```

3. create the development version

```bash
circleci orb publish k8s.yml purple-technology/eks-helpers@dev:0.0.1
```

4. promote development version to the production release

```bash
circleci orb publish promote purple-technology/eks-helpers@dev:0.0.1 patch

```

## EKS specific helpers

### Command configure_aws
Creates an AWS configuration in `~/aws/` with two entries:

- `main` with the actual AWS credentials (supplied by parameters `aws_access_key_id` and `aws_secret_access_key`) and AWS region (supplied by parameter `aws_region`)
- `current` inherits from `main` and it also contains role arn (if supplied by `aws_assume_role_arn`)

### Command aws_decrypt
Decrypts encrypted YAML or JSON with [SOPS](https://github.com/mozilla/sops). 
It requires AWS configuration since the encryption is done via AWS Key Management Service. Creation of AWS config file is handled with `configure_aws` command. 

Command automatically uses `current` profile with assume role. Specify `aws_profile: main` if you don't need this functionality.

**Example with `current` profile:**

```yaml
steps:
    - aws_decrypt:
        input_file_path: << parameters.values_prefix >>/secret.enc.yaml
        output_file_path: << parameters.values_prefix >>/secret.yaml
```

**Example with `main` profile:**

```yaml
steps:
    - aws_decrypt:
        input_file_path: << parameters.values_prefix >>/secret.enc.yaml
        output_file_path: << parameters.values_prefix >>/secret.yaml
        aws_profile: main
```

If you're not happy with the default SOPS version (3.3.0) you can even specify 
custom release url with parameter `sops_release_url`.

### Job build_docker
builds Docker image from the specified Dockerfile (supplied by parameter `docker_dokckerfile`) and puts packed Docker image to the Circle CI workspace. 
Built image is named and tagged with supplied parameters `docker_image_name` and 
`docker_image_tag`

```yaml
steps:
    - eks-helpers/build_docker:
        docker_dockerfile: Dockerfile
        docker_image_name: *docker_image_name
        docker_image_tag: $CIRCLE_SHA1
```

### Job push_ecr
pushes Docker image previously persisted to Circle CI workspace to the AWS ECR. 
Name of the repository is derived from the `aws_account_id`, `aws_region` and 
image's name. 


```yaml
steps:
    - eks-helpers/push_ecr:
        aws_assume_role_arn: *role_arn_staging_asia
        aws_access_key_id: ${AWS_ACCESS_KEY_ID}
        aws_secret_access_key: ${AWS_SECRET_ACCESS_KEY}
        aws_account_id: *account_id_staging_asia
        aws_region: *region_staging_asia
        docker_image_name: *docker_image_name
        docker_image_tag: $CIRCLE_SHA1
```

### Job obtain_helm_chart
obtains Helm chart from the [s3 repository](https://github.com/hypnoglow/helm-s3). 
Once chart is obtained, it's persisted to the Circle CI workspace so downstream 
jobs can use it to install or upgrade Helm releases.

Plase note that we have only two Helm repositories:

|s3 bucket|AWS account|AWS region|
|---|---|---|
|`s3://purple-tech-helm-prod-fr`|purple-trading|eu-central-1|
|`s3://purple-tech-helm-prod-sg`|axiory|ap-southeast-1|

```yaml
steps:
    - eks-helpers/obtain_helm_chart:
        aws_assume_role_arn: *role_arn_production_asia
        aws_access_key_id: ${AWS_ACCESS_KEY_ID}
        aws_secret_access_key: ${AWS_SECRET_ACCESS_KEY}
        aws_region: *region_production_asia
        helm_chart_version: *helm_chart_version
        helm_chart_name: *helm_chart_name
        helm_s3_bucket: *repository_bucket_asia
```

### Job decrypt_secrets
decrypts SOPS secrets files with command `aws_decrypt` and persists them 
to the Circle CI workspace. Output secrets have always the same path: 
`/tmp/out/secret.yaml`

```yaml
steps:
    - eks-helpers/decrypt_secrets:
        input_file_path: Helm/values/production-asia/secret.enc.yaml
        aws_assume_role_arn: *role_arn_production_asia
        aws_access_key_id: ${AWS_ACCESS_KEY_ID}
        aws_secret_access_key: ${AWS_SECRET_ACCESS_KEY}
        aws_region: *region_production_asia
```

### Job helm_deploy_php_simple
deploys `chart-php` to the EKS cluster. It always requires workspace with persisted 
Helm chart (please see `obtain_helm_chart`) and optionally decrypted secrets 
(please see `decrypt_secrets`).

If you don't need secrets - just skip the `secrets_path` parameter.

**Additional configuration parameters:**

|parameter|type|description|
|---|---|---|
|`kubernetes_namespace_override`|`string`|useful when you want to deploy Helm release to the namespace of your choice|
|`helm_additional_params`|`string`|useful for supplying additional Helm params e.g. `--recreate-pods`|
|`helm_release_suffix`|`string`|use this parameter when you want to distinguish releases from the same git branch e.g. `eu` or `asia`|
|`force_upgrade`|`boolean`|appends `--force` to upgrade command if set to `true`, default value is `false`|
|`preview`|`boolean`|passes value `preview.enabled=true` to the Helm chart, default value is `false`|
|`dry_run`|`boolean`|when set to `true`, it appends `--dry-run` to upgrade account so no changes are made by executing deploy step, default value is `false`|

```yaml
steps:
    - eks-helpers/helm_deploy_php_simple:
        aws_region: *region_production_eu
        aws_access_key_id: ${AWS_ACCESS_KEY_ID}
        aws_secret_access_key: ${AWS_SECRET_ACCESS_KEY}
        aws_assume_role_arn: *role_arn_production_eu
        aws_account_id: *account_id_production_eu
        app_name: *app_name
        values_path: Helm/values/production-eu/main.yaml
        secrets_path: /tmp/out/secret.yaml
        docker_image_nginx_name: *docker_image_nginx_name
        docker_image_nginx_tag: $CIRCLE_SHA1
        docker_image_fpm_name: *docker_image_fpm_name
        docker_image_fpm_tag: $CIRCLE_SHA1
        cluster_name: *cluster_name_production
        helm_chart_name: *helm_chart_name
        helm_chart_version: *helm_chart_version
        dry_run: false
        preview: false
```

### Job helm_deploy_node_simple
deploys `chart-nodejs` to the EKS cluster. It always requires workspace with persisted 
Helm chart (please see `obtain_helm_chart`) and optionally decrypted secrets 
(please see `decrypt_secrets`).

If you don't need secrets - just skip the `secrets_path` parameter.

**Additional configuration parameters:**

|parameter|type|description|
|---|---|---|
|`kubernetes_namespace_override`|`string`|useful when you want to deploy Helm release to the namespace of your choice|
|`helm_additional_params`|`string`|useful for supplying additional Helm params e.g. `--recreate-pods`|
|`helm_release_suffix`|`string`|use this parameter when you want to distinguish releases from the same git branch e.g. `eu` or `asia`|
|`force_upgrade`|`boolean`|appends `--force` to upgrade command if set to `true`, default value is `false`|
|`preview`|`boolean`|passes value `preview.enabled=true` to the Helm chart, default value is `false`|
|`dry_run`|`boolean`|when set to `true`, it appends `--dry-run` to upgrade account so no changes are made by executing deploy step, default value is `false`|
|`helm_release_name_override`|`string`|set when you don't want to use deterministic name convention for the Helm release name|
|`branch_name_override`|`string`|set when you don't want to use the actual branch name in the application hostname e.g. `staging`|

```yaml
steps:
    - eks-helpers/helm_deploy_node_simple:
        requires: ["approve_production"]
        name: deploy_production
        aws_region: *region_production_asia
        aws_access_key_id: ${AWS_ACCESS_KEY_ID}
        aws_secret_access_key: ${AWS_SECRET_ACCESS_KEY}
        aws_assume_role_arn: *role_arn_production_asia
        aws_account_id: *account_id_production_asia
        app_name: *app_name
        values_path: Helm/values/production-asia/main.yaml
        secrets_path: /tmp/out/secret.yaml
        docker_image_name: *docker_image_name
        docker_image_tag: $CIRCLE_SHA1
        cluster_name: *cluster_name_production_asia
        helm_chart_name: *helm_chart_name
        helm_chart_version: *helm_chart_version
        dry_run: false
        preview: false
```

## Example scenarios


