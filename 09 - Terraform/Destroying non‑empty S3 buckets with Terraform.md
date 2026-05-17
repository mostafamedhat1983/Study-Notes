---
tags:
  - Terraform
---

By default, Terraform will fail to destroy an S3 bucket if it still contains objects or versions. When you run `terraform destroy` (or remove the bucket from configuration and apply), AWS returns a `BucketNotEmpty` error if the bucket is not empty.

For the `aws_s3_bucket` resource, Terraform provides a `force_destroy` argument to handle this safely and explicitly.

```hcl
resource "aws_s3_bucket" "logs" {
  bucket = "my-logs-bucket"

  force_destroy = true
}
```

When `force_destroy = false` (the default):

- Terraform tries to delete the bucket.
- If there are any objects (or versions) in the bucket, the delete fails with a `BucketNotEmpty` error.
- You must empty the bucket manually (CLI / console / script) before Terraform can destroy it.

When `force_destroy = true`:

- Terraform will delete all objects (and versions) in the bucket as part of destroying the resource.
- After emptying the bucket, Terraform deletes the bucket itself.
- This is useful for non‑critical buckets such as:
  - logs
  - scratch / temp data
  - test environments
- This removal is permanent: once objects are deleted, they are not recoverable.

Important detail:

- If you add `force_destroy = true` to an existing managed bucket, you must run `terraform apply` first so that Terraform records this change for the resource.
- Only after that apply will a subsequent `terraform destroy` (or a later apply that removes the bucket from configuration) actually empty and delete the bucket automatically.

Rule of thumb:

- Use `force_destroy = true` only on buckets where automatic, irreversible deletion of all objects is acceptable (for example, logs or test data).
- Avoid `force_destroy` on buckets that hold important or production data; instead, empty and decommission them manually with clear approvals.