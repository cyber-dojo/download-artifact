
Composite workflow to download a docker image created by
[reusable-actions-workflow](https://github.com/cyber-dojo/reusable-actions-workflows)/.github/workflows/[secure-docker-build.yml](https://github.com/cyber-dojo/reusable-actions-workflows/blob/main/.github/workflows/secure-docker-build.yml)

Typical use is like this:

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
      CHECKOUT_REPOSITORY: cyber-dojo/saver
      CHECKOUT_REF: ${{ github.sha }}
      CHECKOUT_DEPTH: 1
      IMAGE_NAME: ${{ needs.setup.outputs.ecr_registry }}/${{ needs.setup.outputs.service_name }}
      IMAGE_TAG: ${{ needs.setup.outputs.image_tag }}
      IMAGE_BUILD_ARGS: |
        COMMIT_SHA=${{ github.sha }}
        BASE_IMAGE=${{ inputs.BASE_IMAGE }}
      ATTEST_TO_KOSLI: ${{ github.ref == 'refs/heads/main' }}        
      KOSLI_FLOW: ${{ env.KOSLI_FLOW }}
      KOSLI_TRAIL: ${{ env.KOSLI_TRAIL }}
      KOSLI_REFERENCE_NAME: saver
    secrets:
      KOSLI_API_TOKEN: ${{ secrets.KOSLI_API_TOKEN }}


  after-build-image:
    runs-on: ubuntu-latest
    needs: [setup, build-image]
    env:
      IMAGE_NAME:        ${{ needs.setup.outputs.ecr_registry }}/${{ needs.setup.outputs.service_name }}
      KOSLI_FINGERPRINT: ${{ needs.build-image.outputs.digest }}
    steps:
      - name: Download docker image
        uses: cyber-dojo/download-artifact@main
        with:
          IMAGE_NAME:   ${{ needs.setup.outputs.image_name }}
          IMAGE_DIGEST: ${{ needs.build-image.outputs.digest }}
      ...
```
