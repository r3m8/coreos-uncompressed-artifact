# Fedora CoreOS OpenStack — Uncompressed Image

A tiny GitHub repo that publishes the current **Fedora CoreOS stable** OpenStack image as an
**uncompressed** `qcow2`, downloadable as a public GitHub **Release asset** (no login required).

## Why this project exists

Fedora CoreOS only publishes its OpenStack image as an xz-compressed qcow2, for example
`fedora-coreos-<version>-openstack.x86_64.qcow2.xz`. There is **no official uncompressed
artifact**.

The Terraform OpenStack provider cannot take advantage of the upstream compressed image
efficiently. From the provider source (cloned here in `terraform-provider-openstack/`):

- `openstack/resource_openstack_images_image_v2.go` declares `web_download` and `decompress`
  as mutually exclusive (`ConflictsWith`, lines 169–181). You cannot ask Glance to download
  the image and decompress it at the same time.
- With `web_download = true`, the provider calls Glance's `web-download` import method
  (`imageimport.Create`, lines 276–291). Glance fetches the URL **as-is** — it never
  decompresses, so a `.xz` would end up stored in Glance as a compressed file.
- Without `web_download`, the provider downloads the image to a local cache and optionally
  decompresses it client-side (`images_image_v2.go`, `resourceImagesImageV2File`,
  lines 96–201), then uploads the whole file to Glance (`imagedata.Upload`). For a ~1 GB
  compressed / ~2 GB uncompressed image that is a slow
  download → extract → upload round-trip.

**This project removes that problem:** a daily GitHub Actions workflow downloads the latest
stable release, decompresses it, verifies checksums, and publishes the **uncompressed** qcow2
as a release asset. Terraform can then use `web_download = true` straight against the
uncompressed file, letting Glance import it directly — fast, no local round-trip.

## How it works

A scheduled workflow (daily at 06:17 UTC, plus a manual `workflow_dispatch` trigger with a
`force` option) runs on a free `ubuntu-26.04` public runner and:

1. Resolves the latest stable version from
   `https://builds.coreos.fedoraproject.org/prod/streams/stable/releases.json`
   (with a fallback to the documented stream metadata endpoint).
2. Reads the per-release `release.json` for the `openstack` `qcow2.xz` artifact:
   its `location`, `sha256` (compressed) and `uncompressed-sha256`.
3. Skips the run if that version is already published.
4. Downloads the `.xz`, verifies `sha256`.
5. Decompresses with `xz -d -T0`, verifies `uncompressed-sha256`.
6. Publishes to a rolling GitHub release and uploads the two assets.

## Download

Permanent "latest" URLs (anonymous, no login):

- Image: `https://github.com/<owner>/<repo>/releases/latest/download/fedora-coreos-openstack.x86_64.qcow2`
- Checksums + version: `https://github.com/<owner>/<repo>/releases/latest/download/fedora-coreos-openstack.x86_64.qcow2.sha256`

Or with the GitHub CLI:

```bash
gh release download -R <owner>/<repo> --pattern "fedora-coreos-openstack.x86_64.qcow2*"
```

### Verify

```bash
# version + hashes are in the .sha256 metadata file
cat fedora-coreos-openstack.x86_64.qcow2.sha256

# verify the image against the uncompressed-sha256 value
echo "<uncompressed-sha256>  fedora-coreos-openstack.x86_64.qcow2" | sha256sum -c -
```

## Using with Terraform

Point the provider's `web_download` straight at the uncompressed asset:

```hcl
resource "openstack_images_image_v2" "fedora_coreos" {
  name             = "fedora-coreos"
  image_source_url = "https://github.com/<owner>/<repo>/releases/latest/download/fedora-coreos-openstack.x86_64.qcow2"
  web_download     = true

  container_format = "bare"
  disk_format      = "qcow2"
}
```

Because the asset is uncompressed, Glance imports it directly — no `decompress` needed (and
`decompress` cannot be combined with `web_download` anyway).

## Notes & caveats

- **The repository needs at least one commit before the first run.** GitHub creates the
  release tag from the default branch; an empty repo has no commit to tag.
- **"Release immutability" must be disabled** on the repository. If enabled, `gh release edit`
  and `gh release upload --clobber` are blocked.
- The qcow2 is ~1.93 GiB (just under GitHub's 2 GiB per-file limit). It is replaced in place
  on each update, so the permanent URL keeps working.
- `/releases/latest/download/<asset>` occasionally serves a cached older asset (known GitHub
  quirk). If you must pin exactly, use the versioned release page
  (`.../releases/tag/fedora-coreos-openstack`) instead.
- GitHub **disables scheduled workflows after ~60 days** without activity on the default
  branch; push any commit to re-enable the cron.
- Free GitHub-hosted runners and storage are used: keep the repository **public** so usage is
  free.
