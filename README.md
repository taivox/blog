# Mitigating Docker Hub rate limitations and Crossplane package restrictions with registry proxies

## The challenge

Imagine your production Kubernetes cluster grinding to a halt because you've exceeded 10 image pulls in an hour. That's the reality we faced when Docker Hub announced their new rate limits in early 2025. More details can be found in [Docker's official documentation](https://docs.docker.com/docker-hub/usage/).

Around the same time, Upbound announced a policy restricting free users to pulling only the latest Crossplane package versions as of March 25th, limiting access to older versions that many production workflows depend upon. The policy is documented in [Upbound's official documentation](https://docs.upbound.io/providers/policies/#access).

While Docker temporarily paused their policy, smart teams are preparing now rather than scrambling when it inevitably returns. The Upbound restrictions remain in effect, and who knows what's next?

If you're running Kubernetes with frequent scaling and spot instances like our clients do, these changes meant significant operational risks. We ran the numbers and realized our clients would inevitably exceed these limits during standard operations:

- A cluster with 10 nodes requiring 10 images each = 100 pulls.
- Node replacements resulting from cluster upgrades, scaling operations or spot instance recycling = additional pulls for new nodes.
- GitLab CI/CD pipelines on Kubernetes runners = further pulls against the quota.

## Our approach

We sat down and asked ourselves: what do we actually need to solve this?

1. Implement a workaround for the newly imposed rate limitations from Docker Hub.
2. Ensure continuous access to specific Crossplane package versions regardless of upstream restrictions.
3. Enhance infrastructure resilience against external registry service disruptions.
4. Develop a solution that can be implemented across multiple cloud providers.

Whatever we built had to be invisible to applications and easy for infrastructure teams to manage.

### Why not just pay for Docker Hub and Upbound?

We considered getting paid accounts for both services, but that wouldn't defend us from registry outages - when Docker Hub or Upbound goes down, paid customers go down with them. Plus, the costs would be significant across our clients with multiple clusters each. We needed something that gave us full control and true resilience.

## The solution: registry proxies

We built registry proxies - smart middlemen that sit between your infrastructure and external registries. These proxies cache images locally after the first pull, effectively eliminating rate limits for subsequent requests.

Here's what you get with this approach:

1. **Rate limit circumvention** - All subsequent pulls come from the local proxy, not directly from Docker Hub.
2. **Enhanced resilience** - Infrastructure continues operating even during external registry outages.
3. **Version preservation** - Critical services maintain access to specific Crossplane package versions despite upstream restrictions.

### AWS ECR pull-through cache implementation example

For AWS environments, we used Amazon ECR's pull-through cache functionality to automatically retrieve and store images from upstream registries upon initial request:

```terraform

variable "example" {
  type = string
  default = "hello-world"
}

```

### Google Artifact Registry remote repository configuration example

For Google Cloud environments, we configured Artifact Registry to establish remote repositories functioning as proxies for Docker Hub:

```terraform
# Export environment variables
# export GOOGLE_PROJECT=<replace-with-your-project-id>
# export GOOGLE_REGION=<replace-with-your-region>
# export GOOGLE_ZONE=<replace-with-your-zone>

# Provider
terraform {
  required_version = ">= 1.5"
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "6.36.0"
    }
  }
}

# Variables
variable "hub_username_secret" {
  description = "Secret Manager secret name for Docker Hub username"
  type        = string
  default     = "gar-proxy-hub-username"
}

variable "hub_access_token_secret" {
  description = "Secret Manager secret name for Docker Hub access token"
  type        = string
  default     = "gar-proxy-hub-access-token"
}

variable "ghcr_username_secret" {
  description = "Secret Manager secret name for GitHub Container Registry username"
  type        = string
  default     = "gar-proxy-ghcr-username"
}

variable "ghcr_access_token_secret" {
  description = "Secret Manager secret name for GitHub Container Registry access token"
  type        = string
  default     = "gar-proxy-ghcr-access-token"
}

# Data
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

# Locals
locals {
  registries = {
    hub-proxy  = { uri = "registry-1.docker.io", username_secret = var.hub_username_secret, access_token_secret = var.hub_access_token_secret, description = "Docker Hub proxy" }
    ghcr-proxy = { uri = "ghcr.io", username_secret = var.ghcr_username_secret, access_token_secret = var.ghcr_access_token_secret, description = "GitHub Container Registry proxy" }
  }
  registries_with_credentials = { for k, v in local.registries : k => v if v.username_secret != "" && v.access_token_secret != "" }

  registry_url_format = "${data.google_client_config.this.region}-docker.pkg.dev/${data.google_client_config.this.project}/%s/"

  registry_urls = { for k, v in google_artifact_registry_repository.gar_proxy : k => format(local.registry_url_format, v.repository_id) }
}

# Resources
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
  description   = each.value.description
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

  cleanup_policies {
    id     = "delete-all-older-than-90d"
    action = "DELETE"
    condition {
      older_than = "90d"
    }
  }
}

# Outputs
output "hub_registry" {
  description = "Docker Hub proxy"
  value       = local.registry_urls["hub-proxy"]
}

output "ghcr_registry" {
  description = "GitHub Container Registry proxy"
  value       = local.registry_urls["ghcr-proxy"]
}

```

We didn't stop at Docker Hub - we added proxies for Quay.io, GitHub Container Registry, and every other registry our clients use. This comprehensive approach protects against rate limits and service disruptions from any registry.

### Helm chart updates

We updated Helm charts to reference container images through our established proxies:

**AWS ECR reference format:**

```
<account-id>.dkr.ecr.<region>.amazonaws.com/hub-proxy/<image>:<tag>
```

**Google Artifact Registry reference format:**

```
<region>-docker.pkg.dev/<project-id>/hub-proxy/<image>:<tag>
```

For Crossplane packages, which are essentially OCI-compliant Docker images, we established equivalent proxy configurations and updated the package references accordingly.

## Outcomes

Our implementation delivered these key benefits:

1. **Eliminated rate limitation impacts** - Kubernetes clusters now retrieve all container images through our proxy infrastructure, registering as a single pull against Docker Hub's quota regardless of the number of nodes requiring the image.

2. **Ensured version accessibility** - We maintained access to required non-latest Crossplane package versions, enabling more controlled upgrade processes.

3. **Improved service resilience** - During Docker Hub outages, our clients' infrastructure continued to operate without disruption, maintaining critical business services.

4. **Registry independence** - By implementing proxies for multiple registries, our clients' infrastructure is now insulated from policy or pricing changes at any single registry provider.

5. **Enhanced performance** - As a bonus, containers started faster because they were pulling images from the local proxy instead of across the internet.

## Conclusion

Rolling out registry proxies across multiple cloud providers required initial effort, but our clients gained something invaluable: infrastructure that keeps running no matter what happens at Docker Hub, Upbound, or any other registry.
