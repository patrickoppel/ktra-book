# Release Packages on Github

### About

The following describes an additional set-up to share your registry with users outside your local and/or server environment.

### Add repository link in Cargo.toml

To enable `Ktra` to download the `.crate` file later, the repository needs to be added in the `Cargo.toml` file before publishing. This adds the repository to the metadata shared in your registry repository.

### Add github workflow to your crate

Add the following workflow to your crate under `.github/workflows/release.yml` to automatically generate a release and append the `.crate` file every time you push to your repository. This will only create a new release if the version number has changed, and panic otherwise.

It doesn't matter if this is done before or after you publish your crate.

TODO!!! 
Test with private crate

```
name: Release

on:
  push:
    branches:
      - main # Trigger workflow on push to the main branch

jobs:
  release:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code
      uses: actions/checkout@v2

    - name: Setup Rust
      uses: actions-rs/toolchain@v1
      with:
        toolchain: stable
        override: true   
    
    - name: Build and test
      run: |
        cargo build --release
        cargo test

    - name: package
      run: cargo package

    - name: Get crate info
      id: crate_info
      run: |
        echo "::set-output name=VERSION::$(cargo read-manifest | jq -r .version)"
        echo "::set-output name=NAME::$(cargo read-manifest | jq -r .name)"

    - name: Upload package
      uses: actions/upload-artifact@v2
      with:
        name: ${{ steps.crate_info.outputs.NAME }}-${{ steps.crate_info.outputs.VERSION }}
        path: target/package/*.crate
    
    - name: Get version
      id: get_version
      run: |
        VERSION=$(grep "^version" Cargo.toml | cut -d' ' -f3 | tr -d '"')
        echo "::set-output name=VERSION::${VERSION}"

    - name: Check if tag exists
      id: check_tag
      run: |
        TAG_NAME="v${{ steps.get_version.outputs.VERSION }}"
        HTTP_STATUS=$(curl --silent --head --location "https://github.com/$GITHUB_REPOSITORY/releases/tag/$TAG_NAME" | grep HTTP | cut -d ' ' -f2)
        if [ "$HTTP_STATUS" == "200" ]; then
          echo "::set-output name=EXISTS::true"
        else
          echo "::set-output name=EXISTS::false"
        fi
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

    - name: Create and push tag
      if: steps.check_tag.outputs.EXISTS == 'false'
      run: |
        TAG_NAME="v${{ steps.get_version.outputs.VERSION }}"
        echo "TAG_NAME=${TAG_NAME}" >> $GITHUB_ENV
        git tag $TAG_NAME
        git push origin ${{ env.TAG_NAME }}

    - name: Create Release
      id: create_release
      uses: actions/create-release@v1
      if: steps.check_tag.outputs.EXISTS == 'false'
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      with:
        tag_name: ${{ steps.get_version.outputs.VERSION }}
        release_name: Release ${{ steps.get_version.outputs.VERSION }}
        draft: false
        prerelease: false

    - name: Upload Release Asset
      id: upload-release-asset 
      uses: actions/upload-release-asset@v1
      if: steps.check_tag.outputs.EXISTS == 'false'
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      with:
        upload_url: ${{ steps.create_release.outputs.upload_url }}
        asset_path: target/package/${{ steps.crate_info.outputs.NAME }}-${{ steps.crate_info.outputs.VERSION }}.crate
        asset_name: ${{ steps.crate_info.outputs.NAME }}-${{ steps.crate_info.outputs.VERSION }}.crate
        asset_content_type: application/octet-stream
```