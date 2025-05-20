# Dodge Docker's Rate Limits: Set Up Registry Proxies in Minutes

Docker Hub's rate limiting (100 pulls/6 hours) can cripple Kubernetes clusters with frequent scaling or spot instances. Similarly, Upbound's Crossplane packages face version access restrictions. Registry proxies offer an elegant solution.

## AWS ECR Pull-Through Cache

```terraform
resource "aws_secretsmanager_secret" "docker_hub_creds" {
  name = "ecr-pullthroughcache/docker-hub"
}

resource "aws_secretsmanager_secret_version" "docker_hub_creds_version" {
  secret_id     = aws_secretsmanager_secret.docker_hub_creds.id
  secret_string = jsonencode({
    username    = var.docker_hub_username
    accessToken = var.docker_hub_access_token
  })
}

resource "aws_ecr_pull_through_cache_rule" "docker_hub" {
  ecr_repository_prefix = "docker-hub"
  upstream_registry_url = "registry-1.docker.io"
  credential_arn        = aws_secretsmanager_secret.docker_hub_creds.arn
}
```

## Google Artifact Registry Proxy

```terraform
resource "google_artifact_registry_repository" "docker_hub_proxy" {
  location      = var.region
  repository_id = "docker-hub-proxy"
  format        = "DOCKER"
  mode          = "REMOTE_REPOSITORY"
  
  remote_repository_config {
    description = "Docker Hub proxy"
    docker_repository {
      public_repository = "DOCKER_HUB"
    }
    upstream_credentials {
      username               = var.docker_hub_username
      password_secret_version = google_secret_manager_secret.docker_hub_token.id
    }
  }
}
```

After setup, just reference images with your proxy path:
- AWS: `account-id.dkr.ecr.region.amazonaws.com/docker-hub/nginx:latest`
- GCP: `region-docker.pkg.dev/project-id/docker-hub-proxy/nginx:latest`

These proxies cache images after first pull, eliminate rate limits on subsequent pulls, and provide faster image retrieval for your scaling Kubernetes nodes. They also help bypass Crossplane's restrictive package access policies.