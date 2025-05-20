## The Growing Registry Challenge

In early 2025, Docker Hub announced a significant change: free users would be limited to just 10 image pulls per hour starting April 1st. Around the same time, Upbound revealed that free users could only pull the latest Crossplane package versions as of March 25th.

For Kubernetes environments with frequent scaling and spot instances, these restrictions create a serious operational risk. A single node scaling event could exhaust your pull quota almost instantly, leaving your infrastructure unable to pull images for hours.

Why is this happening? Container registries face increasing costs as free usage grows. While these policy changes help providers remain sustainable, they create new challenges for development teams who need to maintain reliable infrastructure.

## Conclusion

While Docker Hub and Upbound's policy changes present challenges, they also create an opportunity to improve your infrastructure's resilience. Registry proxies not only help you avoid rate limits but also enhance performance, control, and reliability.

For our Kubernetes clients with frequent scaling and spot instances, implementing registry proxies across AWS ECR, Google Artifact Registry, and Harbor has been transformative. We now sleep better knowing their infrastructure doesn't depend on external registry availability or rate limits.

Setting up these proxies might take some initial effort, but the long-term operational benefits make it well worth the investment. Don't wait until rate limits become a production issue – start implementing your registry proxy strategy today!

_This post was written by the Platform Engineering team at Entigo, building resilient infrastructure for modern applications._# Dodge Docker's Rate Limits: Set Up Registry Proxies in Minutes

## The Solution: Registry Proxies

Registry proxies act as intermediaries between your infrastructure and external image repositories. They cache images locally after the first pull, eliminating rate limits for subsequent requests. This provides three key benefits:

1. **Bypass rate limits** - All pulls come from your proxy, not directly from Docker Hub
2. **Improved resilience** - Your infrastructure continues working even during external registry outages
3. **Version preservation** - Access specific Crossplane package versions even after restrictions

Let's look at how to implement these registry proxies using Terraform.

## AWS ECR Pull-Through Cache

Amazon ECR's pull-through cache feature allows you to create rules that automatically pull and cache images from upstream registries. When a container tries to pull an image through ECR that isn't cached yet, ECR pulls it from the upstream registry, caches it, and returns it to the requester.

```terraform
# First, create a secret to store Docker Hub credentials
resource "aws_secretsmanager_secret" "docker_hub_creds" {
  name = "ecr-pullthroughcache/docker-hub"
}

# Store the Docker Hub username and access token
resource "aws_secretsmanager_secret_version" "docker_hub_creds_version" {
  secret_id     = aws_secretsmanager_secret.docker_hub_creds.id
  secret_string = jsonencode({
    username    = var.docker_hub_username
    accessToken = var.docker_hub_access_token
  })
}

# Create the pull-through cache rule
resource "aws_ecr_pull_through_cache_rule" "docker_hub" {
  ecr_repository_prefix = "docker-hub"
  upstream_registry_url = "registry-1.docker.io"
  credential_arn        = aws_secretsmanager_secret.docker_hub_creds.arn
}
```

This configuration creates a cached proxy for Docker Hub in your AWS ECR registry. The prefix "docker-hub" will appear in your ECR repository path, making it clear which images are being proxied.

## Google Artifact Registry Proxy

Google Cloud offers similar functionality through Artifact Registry's remote repositories. This feature allows you to create a proxy that pulls and caches images from external sources like Docker Hub.

```terraform
# Create a remote repository in Google Artifact Registry
resource "google_artifact_registry_repository" "docker_hub_proxy" {
  location      = var.region
  repository_id = "docker-hub-proxy"
  format        = "DOCKER"
  mode          = "REMOTE_REPOSITORY"

  # Configure the remote repository to point to Docker Hub
  remote_repository_config {
    description = "Docker Hub proxy"
    docker_repository {
      public_repository = "DOCKER_HUB"
    }
    # Add Docker Hub credentials for authenticated pulls
    upstream_credentials {
      username               = var.docker_hub_username
      password_secret_version = google_secret_manager_secret.docker_hub_token.id
    }
  }
}
```

This creates a Docker Hub proxy in your Google Cloud project, allowing your applications to pull Docker images through Artifact Registry instead of directly from Docker Hub.

## Using Your Registry Proxies

After setting up these proxies, you'll need to update your image references to use the proxy paths instead of pulling directly from Docker Hub. Here's how to reference images through your proxies:

### For AWS ECR:

```
account-id.dkr.ecr.region.amazonaws.com/docker-hub/nginx:latest
```

### For Google Artifact Registry:

```
region-docker.pkg.dev/project-id/docker-hub-proxy/nginx:latest
```

For Kubernetes clusters, you can update your deployment manifests or use image pull secrets to authenticate with your private registry. If you're using Helm charts, you'll need to update the image repository values to point to your proxy.

## Benefits Beyond Rate Limiting

Registry proxies provide advantages that extend far beyond just avoiding rate limits:

1. **Faster deployments** - Cached images pull more quickly from your local registry
2. **Reduced bandwidth costs** - Pull images once instead of multiple times across nodes
3. **Greater control** - Audit and govern which images are being used in your environment
4. **Version locking** - Protect against upstream changes or removals of images

For Crossplane users, this approach is particularly valuable, as it preserves access to specific package versions that might otherwise become inaccessible due to Upbound's new policies.

## Implementation Considerations

When implementing registry proxies, consider these important factors:

- **Authentication**: Ensure your proxies have the necessary credentials to access private repositories in the upstream registry
- **Storage costs**: Cached images consume storage space, so monitor usage and implement lifecycle policies if needed
- **Image scanning**: Configure vulnerability scanning on your proxy registry to maintain security
- **Multiple registries**: Consider creating proxies for other common registries besides Docker Hub (like Quay.io or GitHub Container Registry)
## Implementation Considerations

When implementing registry proxies, consider these important factors:

- **Authentication**: Ensure your proxies have the necessary credentials to access private repositories in the upstream registry
- **Storage costs**: Cached images consume storage space, so monitor usage and implement lifecycle policies if needed
- **Image scanning**: Configure vulnerability scanning on your proxy registry to maintain security
- **Multiple registries**: Consider creating proxies for other common registries besides Docker Hub (like Quay.io or GitHub Container Registry)
