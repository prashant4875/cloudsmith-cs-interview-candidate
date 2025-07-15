## Issues in existing Code
1. Build step is missing in build_package.yml. Adding below build step.
   - name: Build package
        run: python -m build
     
2. Name and Version is mising in pyproject.toml file. Added below configuration in pyproject.toml.
  [project]
  name = "example_package"
  dynamic = ["version"]

  [tool.hatch.version]
  path = "./src/example_package/__init__.py"
  
3. Enable OIDC in Cloudsmith Namespace for Github and add OIDC slug in Github environment varaibles.
   
4. In release_package.yml, cloudsmith CLI is not installed. Hence installed it with OIDC authentication.
   
    - name: Install Cloudsmith CLI
        uses: cloudsmith-io/cloudsmith-cli-action@v1.0.1
        with:
          oidc-namespace: ${{ env.CLOUDSMITH_NAMESPACE }}
          oidc-service-slug: ${{ env.CLOUDSMITH_SERVICE_SLUG }}

5. Id-token write permission is missing in release_package.yml. Added it to install CLI and push package to staging repo by OIDC.
   permissions:
    id-token: write
    contents: read
    actions: read

6. In promote_package.yml, repo entries are wrong. Fixed by passing correct repo name.
   CLOUDSMITH_STAGING_REPO: 'staging'
   CLOUDSMITH_PRODUCTION_REPO: 'prod'

## Changes made to run promote_package.yml workflow

1. Create Webhook in Cloudsmith Staging repo with below JSON payload.
   {
      "event_type": "package_synchronized"
    }
   
2. Update event trigger with below action.
   on:
    repository_dispatch:
      types: [package_synchronized]

3. Update steps in promote_package.yml to add tag and move artifact to prod repo.
   
      - name: Find latest valid package in staging
           id: latest
           run: |
             echo "Querying latest valid package in staging repository..."
   
             PACKAGE_DATA=$(cloudsmith list package \
               ${{ env.CLOUDSMITH_NAMESPACE }}/${{ env.CLOUDSMITH_STAGING_REPO }} \
               -F json)
   
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
   
         - name: Tag latest package as ready-for-production
           run: |
             IDENTIFIER="${{ steps.latest.outputs.identifier }}"
             PACKAGE_NAME="${{ steps.latest.outputs.package_name }}"
             VERSION="${{ steps.latest.outputs.version }}"
   
             echo "🏷️ Tagging latest package [$PACKAGE_NAME:$VERSION] with 'ready-for-production' tag."
             cloudsmith tags add \
               ${{ env.CLOUDSMITH_NAMESPACE }}/${{ env.CLOUDSMITH_STAGING_REPO }}/$IDENTIFIER \
               ready-for-production
   
         - name: Promote all ready-for-production packages
           run: |
             echo "🔍 Finding all packages tagged with ready-for-production..."
   
             PACKAGE_DATA=$(cloudsmith list package \
               ${{ env.CLOUDSMITH_NAMESPACE }}/${{ env.CLOUDSMITH_STAGING_REPO }} \
               -q "tag:ready-for-production" -F json)
   
             PACKAGE_COUNT=$(echo "$PACKAGE_DATA" | jq '[.data[] | select(.version != null and .version != "")] | length')
   
             if [ "$PACKAGE_COUNT" -eq 0 ]; then
               echo "❌ No valid packages found with ready-for-production tag."
               exit 1
             fi
   
             echo "✅ Found $PACKAGE_COUNT package(s) tagged ready-for-production."
   
             for IDENTIFIER in $(echo "$PACKAGE_DATA" | jq -r '.data[] | select(.version != null and .version != "") | .identifier_perm'); do
               echo "🚀 Promoting package: $IDENTIFIER"
               cloudsmith mv --yes \
                 ${{ env.CLOUDSMITH_NAMESPACE }}/${{ env.CLOUDSMITH_STAGING_REPO }}/$IDENTIFIER \
                 ${{ env.CLOUDSMITH_PRODUCTION_REPO }}
             done
   
