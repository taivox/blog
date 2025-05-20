# Mitigating Docker Hub Rate Limitations and Crossplane Package Restrictions with Registry Proxies

## The Challenge

In early 2025, Docker Hub announced a significant policy change: free users would be limited to 10 image pulls per hour starting April 1st. This change introduced substantial constraints for development and production environments worldwide. More details can be found in [Docker's official announcement](https://www.docker.com/blog/revisiting-docker-hub-policies-prioritizing-developer-experience).

At the same time, Upbound implemented a policy restricting free users to pulling only the latest Crossplane package versions as of March 25th, limiting access to older versions that many production workflows depend upon. The policy is documented in [Upbound's official documentation](https://docs.upbound.io/providers/policies/#access).

For Kubernetes-based infrastructure with frequent scaling operations and spot instance usage, these policy changes presented significant operational risks. Our analysis indicated that clients would exceed these limitations within minutes during standard operations:

- A cluster with 20 nodes requiring 5 images each = 100 pulls
- Node replacements resulting from spot instance recycling = additional pulls for new nodes
- GitLab CI/CD pipelines on Kubernetes runners = further pulls against the quota

While Docker Hub had not yet fully enforced their policy, it was clear that proactive measures were necessary to prevent potential disruptions to critical infrastructure services.

## Our Approach

After analyzing the situation, we established several key requirements for an effective solution:

1. Implement a strategy to circumvent the newly imposed rate limitations from Docker Hub
2. Ensure continuous access to specific Crossplane package versions regardless of upstream restrictions
3. Enhance infrastructure resilience against external registry service disruptions
4. Develop a solution that can be implemented across multiple cloud providers

Additionally, any implemented solution needed to maintain transparency to client applications and minimize operational overhead for infrastructure teams.

## The Solution: Registry Proxies

Our engineering team designed and implemented a registry proxy architecture that acts as an intermediary layer between client infrastructure and external image repositories. These proxies cache images locally after the first pull, effectively eliminating rate limits for subsequent requests.

This approach provides three primary benefits:

1. **Rate limit circumvention** - All subsequent pulls come from the local proxy, not directly from Docker Hub
2. **Enhanced resilience** - Infrastructure continues operating even during external registry outages
3. **Version preservation** - Critical services maintain access to specific Crossplane package versions despite upstream restrictions

### AWS ECR Pull-Through Cache Implementation

For AWS environments, we leveraged Amazon ECR's pull-through cache functionality to automatically retrieve and store images from upstream registries upon initial request:

```terraform
# Initialize a secret to store Docker Hub authentication credentials
resource "aws_secretsmanager_secret" "docker_hub_creds" {
  name = "ecr-pullthroughcache/docker-hub"
}

# Store the Docker Hub authentication parameters
resource "aws_secretsmanager_secret_version" "docker_hub_creds_version" {
  secret_id     = aws_secretsmanager_secret.docker_hub_creds.id
  secret_string = jsonencode({
    username    = var.docker_hub_username
    accessToken = var.docker_hub_access_token
  })
}

# Establish the pull-through cache configuration
resource "aws_ecr_pull_through_cache_rule" "docker_hub" {
  ecr_repository_prefix = "docker-hub"
  upstream_registry_url = "registry-1.docker.io"
  credential_arn        = aws_secretsmanager_secret.docker_hub_creds.arn
}
```

### Google Artifact Registry remote repository configuration example

For Google Cloud environments, we configured Artifact Registry to establish remote repositories functioning as proxies for Docker Hub:

```terraform
variable "hub_username_secret" {
  description = "Secret Manager secret name for Docker Hub username"
  type        = string
  default     = ""
}

variable "hub_access_token_secret" {
  description = "Secret Manager secret name for Docker Hub access token"
  type        = string
  default     = ""
}

variable "ghcr_username_secret" {
  description = "Secret Manager secret name for GitHub Container Registry username"
  type        = string
  default     = ""
}

variable "ghcr_access_token_secret" {
  description = "Secret Manager secret name for GitHub Container Registry access token"
  type        = string
  default     = ""
}

data "google_project" "this" {}

data "google_client_config" "this" {}

data "google_secret_manager_secret_version_access" "gar_proxy_username" {
  for_each = local.registries_with_credentials
  secret   = each.value.username_secret
}

data "google_secret_manager_secret_version_access" "gar_proxy_access_token" {
  for_each = local.registries_with_credentials
  secret   = each.value.access_token_secret
}

locals {
  registries = {
    hub  = { uri = "registry-1.docker.io", username_secret = var.hub_username_secret, access_token_secret = var.hub_access_token_secret }
    ghcr = { uri = "ghcr.io", username_secret = var.ghcr_username_secret, access_token_secret = var.ghcr_access_token_secret }
  }
  registries_with_credentials = { for k, v in local.registries : k => v if v.username_secret != "" && v.access_token_secret != "" }
}

resource "google_secret_manager_secret_iam_member" "gar_proxy" {
  for_each  = local.registries_with_credentials
  secret_id = each.value.access_token_secret
  role      = "roles/secretmanager.secretAccessor"
  member    = "serviceAccount:service-${data.google_project.this.number}@gcp-sa-artifactregistry.iam.gserviceaccount.com"
}

resource "google_artifact_registry_repository" "gar_proxy" {
  for_each      = local.registries
  depends_on    = [google_secret_manager_secret_iam_member.gar_proxy]
  repository_id = each.key
  format        = "DOCKER"
  mode          = "REMOTE_REPOSITORY"

  remote_repository_config {
    common_repository {
      uri = "https://${each.value.uri}"
    }

    dynamic "upstream_credentials" {
      for_each = each.value.username_secret != "" && each.value.access_token_secret != "" ? [1] : []
      content {
        username_password_credentials {
          username                = data.google_secret_manager_secret_version_access.gar_proxy_username[each.key].secret_data
          password_secret_version = data.google_secret_manager_secret_version_access.gar_proxy_access_token[each.key].name
        }
      }
    }
  }
}

output "hub_registry" {
  description = "Docker Hub proxy "
  value = "${data.google_client_config.this.region}-docker.pkg.dev/${data.google_client_config.this.project}/${google_artifact_registry_repository.gar_proxy["hub"].repository_id}"
}

output "ghcr_registry" {
  description = "GitHub Container Registry proxy"
  value = "${data.google_client_config.this.region}-docker.pkg.dev/${data.google_client_config.this.project}/${google_artifact_registry_repository.gar_proxy["ghcr"].repository_id}"
}

```

Our implementation extends beyond Docker Hub to include proxies for other critical container registries such as Quay.io, GitHub Container Registry (ghcr.io). This comprehensive approach ensures protection against rate limits and service disruptions.

### Kubernetes Manifest Updates

We updated Helm charts and Kubernetes manifests to reference container images through our established proxies:

**AWS ECR reference format:**

```
<account-id>.dkr.ecr.<region>.amazonaws.com/hub-proxy/<image>:<tag>
```

**Google Artifact Registry reference format:**

```
<region>-docker.pkg.dev/<project-id>/hub-proxy/<image>:<tag>
```

For Crossplane packages, which are essentially OCI-compliant Docker images, we established equivalent proxy configurations and updated the package references accordingly.

## Measured Outcomes

Our registry proxy implementation delivered several business and operational benefits:

1. **Eliminated rate limitation impacts** - Kubernetes clusters now retrieve all container images through our proxy infrastructure, registering as a single pull against Docker Hub's quota regardless of the number of nodes requiring the image.

2. **Ensured version accessibility** - We maintained access to required non-latest Crossplane package versions, enabling more controlled upgrade processes.

3. **Improved service resilience** - During Docker Hub outages, our clients' infrastructure continued to operate without disruption, maintaining critical business services.

4. **Enhanced performance** - We notified a reduction in container initialization times due to localized image retrieval from proxy repositories, improving application startup performance.

5. **Registry independence** - By implementing proxies for multiple registries, our clients' infrastructure is now insulated from policy or pricing changes at any single registry provider.

## Conclusion

While the initial implementation required configuration effort across our clients' multi-cloud environments, the solution has substantially improved operational reliability by eliminating external registry dependencies and rate limitations.

This project demonstrates how proactive infrastructure engineering can transform potential business disruptions into opportunities for enhanced reliability and performance.
