
A "partner" composite workflow to load the docker image created by
[reusable-actions-workflow](https://github.com/cyber-dojo/reusable-actions-workflows)/.github/workflows/[secure-docker-build-from-bare-image.yml](https://github.com/cyber-dojo/reusable-actions-workflows/blob/main/.github/workflows/secure-docker-build-from-bare-image.yml)

Typical use is as below. Note:
- In the `build-image` job, the image is built using cyber-dojo/reusable-actions-workflows/.github/workflows/secure-docker-build-from-bare-image.yml@main, FROM the bare image the `bare-image` job builds
- In each subsequent job:
  - Step 1: download the image using this workflow; cyber-dojo/download-artifact@main
  - Step 2: do-something, which must use this image. It must **NOT** rebuild the image
  - Step 3: `kosli attest` the evidence, using the `fingerprint` returned from the `build-image` job. It is **not** possible to calculate the image digest using `--artifact-type`.

```yml
name: Main

...

jobs:
  setup:
    ...
  
  bare-image:
    uses: cyber-dojo/reusable-actions-workflows/.github/workflows/build-bare-image.yml@main
    with:
      checkout_repository: cyber-dojo/saver
      checkout_ref: ${{ github.sha }}
      artifact_prefix: saver-bare-image

  build-image:
    needs: [setup, bare-image]
    uses: cyber-dojo/reusable-actions-workflows/.github/workflows/secure-docker-build-from-bare-image.yml@main
    with:
      checkout_repository: cyber-dojo/saver
      checkout_ref: ${{ github.sha }}
      checkout_fetch_depth: 1
      image_name: ${{ needs.setup.outputs.ecr_registry }}/${{ needs.setup.outputs.service_name }}
      image_tag: ${{ needs.setup.outputs.image_tag }}
      image_build_args: |
        COMMIT_SHA=${{ github.sha }}
      bare_image_artifact_prefix: saver-bare-image
      bare_image_digest_amd64: ${{ needs.bare-image.outputs.digest_amd64 }}
      bare_image_digest_arm64: ${{ needs.bare-image.outputs.digest_arm64 }}
      kosli_flow: ${{ env.KOSLI_FLOW }}
      kosli_trail: ${{ env.KOSLI_TRAIL }}
      kosli_reference_name: saver
      attest_to_kosli: ${{ github.ref == 'refs/heads/main' }}
    secrets:
      kosli_api_token: ${{ secrets.KOSLI_API_TOKEN }}
      docker_user: ${{ secrets.DOCKER_USER }}
      docker_pass: ${{ secrets.DOCKER_PASS }}


  do-something:
    runs-on: ubuntu-latest
    needs: [build-image]
    env:
      KOSLI_FINGERPRINT: ${{ needs.build-image.outputs.digest }}    
    steps:
      - name: Download docker image
        uses: cyber-dojo/download-artifact@main
        with:
          image_digest: ${{ needs.build-image.outputs.digest }}
      
      - name: Do something
        run: |
          # Run some tests on ${{ needs.build-image.outputs.tagged_image_name }}
          ...
          
      - name: Setup Kosli CLI
        if: ${{ github.ref == 'refs/heads/main' }}
        uses: kosli-dev/setup-cli-action@v3
        with:
          version: ${{ vars.KOSLI_CLI_VERSION }}

      - name: Attest evidence to Kosli
        if: ${{ github.ref == 'refs/heads/main' }}
        run:
          kosli attest ...
- 
```
