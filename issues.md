# 🚀 Cloudsmith Python Package CI/CD with GitHub Actions

This repository automates the build, release, and promotion of Python packages to Cloudsmith using GitHub Actions with OIDC authentication.

---

## 🔧 Issues Fixed in Existing Code

### ✅ Build step missing in `build_package.yml`
Added:
```yaml
- name: Build package
  run: python -m build
```

---

### ✅ Name & Version missing in `pyproject.toml`
Added:
```toml
[project]
name = "example_package"
dynamic = ["version"]

[tool.hatch.version]
path = "./src/example_package/__init__.py"
```

---

### ✅ OIDC for Cloudsmith & GitHub
- Enabled OIDC in the Cloudsmith namespace.
- Added OIDC slug to GitHub environment variables.

---

### ✅ Cloudsmith CLI missing in `release_package.yml`
Installed with OIDC authentication:
```yaml
- name: Install Cloudsmith CLI
  uses: cloudsmith-io/cloudsmith-cli-action@v1.0.1
  with:
    oidc-namespace: ${{ env.CLOUDSMITH_NAMESPACE }}
    oidc-service-slug: ${{ env.CLOUDSMITH_SERVICE_SLUG }}
```

---

### ✅ Missing `id-token` permission in `release_package.yml`
Added:
```yaml
permissions:
  id-token: write
  contents: read
  actions: read
```

---

### ✅ Incorrect repository names in `promote_package.yml`
Fixed:
```yaml
CLOUDSMITH_STAGING_REPO: 'staging'
CLOUDSMITH_PRODUCTION_REPO: 'prod'
```

---

## 🚀 Changes Made for `promote_package.yml` Workflow

### 1️⃣ Create a Webhook in Cloudsmith
In the **staging** repository, create a webhook with payload:
```json
{
  "event_type": "package_synchronized"
}
```

---

### 2️⃣ Update Trigger in `promote_package.yml`
```yaml
on:
  repository_dispatch:
    types: [package_synchronized]
```

---

### 3️⃣ Add Steps to Tag & Promote Artifacts

#### 🔍 Find Latest Valid Package
```yaml
- name: Find latest valid package in staging
  id: latest
  run: |
    echo "Querying latest valid package in staging repository..."

    PACKAGE_DATA=$(cloudsmith list package       ${{ env.CLOUDSMITH_NAMESPACE }}/${{ env.CLOUDSMITH_STAGING_REPO }}       -F json)

    # Filter only versioned packages
    IDENTIFIER=$(echo "$PACKAGE_DATA" | jq -r '.data[] | select(.version != null and .version != "") | .identifier_perm' | head -n1)
    PACKAGE_NAME=$(echo "$PACKAGE_DATA" | jq -r '.data[] | select(.version != null and .version != "") | .name' | head -n1)
    VERSION=$(echo "$PACKAGE_DATA" | jq -r '.data[] | select(.version != null and .version != "") | .version' | head -n1)

    if [ -z "$IDENTIFIER" ] || [ "$IDENTIFIER" = "null" ]; then
      echo "❌ No valid versioned package found in staging."
      exit 1
    fi

    echo "✅ Latest valid package: $PACKAGE_NAME:$VERSION"
    echo "identifier=$IDENTIFIER" >> $GITHUB_OUTPUT
    echo "package_name=$PACKAGE_NAME" >> $GITHUB_OUTPUT
    echo "version=$VERSION" >> $GITHUB_OUTPUT
```

---

#### 🏷️ Tag Package as `ready-for-production`
```yaml
- name: Tag latest package as ready-for-production
  run: |
    IDENTIFIER="${{ steps.latest.outputs.identifier }}"
    PACKAGE_NAME="${{ steps.latest.outputs.package_name }}"
    VERSION="${{ steps.latest.outputs.version }}"

    echo "🏷️ Tagging latest package [$PACKAGE_NAME:$VERSION] with 'ready-for-production' tag."
    cloudsmith tags add       ${{ env.CLOUDSMITH_NAMESPACE }}/${{ env.CLOUDSMITH_STAGING_REPO }}/$IDENTIFIER       ready-for-production
```

---

#### 🚀 Promote Tagged Packages
```yaml
- name: Promote all ready-for-production packages
  run: |
    echo "🔍 Finding all packages tagged with ready-for-production..."

    PACKAGE_DATA=$(cloudsmith list package       ${{ env.CLOUDSMITH_NAMESPACE }}/${{ env.CLOUDSMITH_STAGING_REPO }}       -q "tag:ready-for-production" -F json)

    PACKAGE_COUNT=$(echo "$PACKAGE_DATA" | jq '[.data[] | select(.version != null and .version != "")] | length')

    if [ "$PACKAGE_COUNT" -eq 0 ]; then
      echo "❌ No valid packages found with ready-for-production tag."
      exit 1
    fi

    echo "✅ Found $PACKAGE_COUNT package(s) tagged ready-for-production."

    for IDENTIFIER in $(echo "$PACKAGE_DATA" | jq -r '.data[] | select(.version != null and .version != "") | .identifier_perm'); do
      echo "🚀 Promoting package: $IDENTIFIER"
      cloudsmith mv --yes         ${{ env.CLOUDSMITH_NAMESPACE }}/${{ env.CLOUDSMITH_STAGING_REPO }}/$IDENTIFIER         ${{ env.CLOUDSMITH_PRODUCTION_REPO }}
    done
```

---
