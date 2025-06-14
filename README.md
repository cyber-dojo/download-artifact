
A "partner" composite workflow to `docker load` the docker image in a tar file created by
[reusable-actions-workflow](https://github.com/cyber-dojo/reusable-actions-workflows)/.github/workflows/[secure-docker-build.yml](https://github.com/cyber-dojo/reusable-actions-workflows/blob/main/.github/workflows/secure-docker-build.yml)

Typical use is as below. Note:
- The image is built using cyber-dojo/reusable-actions-workflows/.github/workflows/secure-docker-build.yml@main
- The image is downloaded using this workflow; cyber-dojo/download-artifact@main
- In the `Do something` job(s), use this image. Do **NOT** rebuild the image
- In the `kosli attest` commands, use the `fingerprint` returned from the `build-image` job.
It is **not** possible to calculate the image digest using `--artifact-type`.

```yml
name: Main

...

jobs:
  setup:
    ...
  
  build-image:
    needs: [setup]    
    uses: cyber-dojo/reusable-actions-workflows/.github/workflows/secure-docker-build.yml@main
    with:
      checkout_repository: cyber-dojo/saver
      checkout_ref: ${{ github.sha }}
      checkout_fetch_depth: 1
      image_name: ${{ needs.setup.outputs.ecr_registry }}/${{ needs.setup.outputs.service_name }}
      image_tag: ${{ needs.setup.outputs.image_tag }}
      image_build_args: |
        COMMIT_SHA=${{ github.sha }}
        BASE_IMAGE=${{ inputs.BASE_IMAGE }}
      kosli_flow: ${{ env.KOSLI_FLOW }}
      kosli_trail: ${{ env.KOSLI_TRAIL }}
      kosli_reference_name: saver
      attest_to_kosli: ${{ github.ref == 'refs/heads/main' }}        
    secrets:
      kosli_api_token: ${{ secrets.KOSLI_API_TOKEN }}


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
        if: ${{ github.ref == 'refs/heads/main' && (success() || failure()) }}
        uses: kosli-dev/setup-cli-action@v2
        with:
          version: ${{ vars.KOSLI_CLI_VERSION }}

      - name: Attest evidence to Kosli
        if: ${{ github.ref == 'refs/heads/main' && (success() || failure()) }}
        run:
          kosli attest ...
- 
```
