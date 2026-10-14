# Fedora CoreOS OpenStack Uncompressed Image

Publishes every day the latest Fedora CoreOS stable OpenStack image as an uncompressed `qcow2`, downloadable as a public GitHub Release asset. Current limit of Github Release assets is 2 GB; I may migrate to OVH S3 public bucket in the future if the image size will go further (Fedora 45 coming soon).

## Why

Fedora CoreOS only publishes its OpenStack image as an xz-compressed qcow2. There is no official uncompressed artifact (why Redhat, why ?).

The Terraform OpenStack provider **cannot take advantage of the upstream compressed image** efficiently. From the current provider source:

- `openstack/resource_openstack_images_image_v2.go` declares `web_download` and `decompress` as mutually exclusive (`ConflictsWith`). You cannot ask Glance to download the image and decompress it at the same time.
- With `web_download = true`, the provider calls Glance's `web-download` import method (`imageimport.Create`). Glance fetches the URL **as-is** it never decompresses, so a `.xz` would end up stored in Glance as a compressed file.
- Without `web_download`, the provider downloads the image to a local cache and optionally decompresses it client-side (`images_image_v2.go`, `resourceImagesImageV2File`), then uploads the whole file to Glance (`imagedata.Upload`).

This project removes that problem: a daily GitHub Actions workflow downloads the latest stable release, decompresses it, verifies checksums, and publishes the **uncompressed** qcow2 as a release asset. Terraform can then use `web_download = true` straight against the uncompressed file, letting Glance import it directly fast, no local round-trip (where terraform/opentofu is executed).

## Download

- Image: `https://github.com/r3m8/coreos-uncompressed-artifact/releases/latest/download/fedora-coreos-openstack.x86_64.qcow2`
- Checksums + version: `https://github.com/r3m8/coreos-uncompressed-artifact/releases/latest/download/fedora-coreos-openstack.x86_64.qcow2.sha256`

## Using with terraform-provider-openstack

Point the provider's `web_download` straight at the uncompressed asset:

```hcl
resource "openstack_images_image_v2" "fedora_coreos" {
  name             = "fedora-coreos"
  image_source_url = "https://github.com/r3m8/coreos-uncompressed-artifact/releases/latest/download/fedora-coreos-openstack.x86_64.qcow2"
  web_download     = true

  container_format = "bare"
  disk_format      = "qcow2"
}
```