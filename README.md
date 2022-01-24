# Zowe CLI Standalone Package Pipeline

This repository pulls the latest Zowe CLI and Zowe CLI Plugin Artifacts from Artifactory, and generates a standalone ZIP file containing the contents.

The standalone ZIP is intended to be offline-installable for those users who are unable to connect to public NPM registries. The Zowe CLI DB2 Plugin is the lone exception, as this plugin has dependencies on ODBC drivers and DDB licenses which much be obtained by users separately. See the official Zowe Documentation for more information.

## Steps prior to a Zowe Release

Before every Zowe release goes GA, we need to align with the code freeze dates and produce snapshots to be promoted to RCs later in the process. Follow the steps below to accomplish that.

1. Create a PR into `master` with the following changes and information:
    - Update release number to match the RC that's going out (e.g. 1.22.0)
    - Update any packages that changed after the last Zowe Release.
    - Add a summary of the changes in the PR description like below:
      ```yaml
      zowe-cli:
        imperative: 4.13.0 -> 4.13.1
        perf-timing: 1.0.7
        cli: 6.31.0 -> 6.31.1
      zowe-plugins:
        cics: 4.0.2
        db2: 4.1.0
        ims: 2.0.1
        mq: 2.0.1
        secure-credential-store: 4.1.3 -> 4.1.5
        zos-ftp: 1.4.1 -> 1.6.0
      zowe-sdk:
        core: 6.31.0 -> 6.31.1
        provisioning: 6.31.0 -> 6.31.1
        zos-console: 6.31.0 -> 6.31.1
        zos-files: 6.31.0 -> 6.31.1
        zos-jobs: 6.31.0 -> 6.31.1
        zos-tso: 6.31.0 -> 6.31.1
        zos-uss: 6.30.0 -> 6.31.1
        zos-workflows: 6.31.0 -> 6.31.1
        zosmf: 6.31.0 -> 6.31.1
        ```
2. Check to see if the `master` build failed due to missing licenses.
    - If so, rerun the workflow and change the license URL to a previous version
3. Notify `Tom Zhang` on Slack and make sure to copy a link to the PR and the build that uploaded the bundles to `libs-snapshot-local` on JFrog Artifactory
4. After the RC is approved and the license zip is available, we need to update the licenses URL to the Zowe version that's about to go GA.
5. After the Zowe Release is GA, create a new branch with the version corresponding to that of the Zowe Release.

## Nightly Deployment

This repository contains a GitHub workflow that runs every night to mirror @zowe-scoped packages from Artifactory to npmjs.org. The list of packages to deploy is defined in zowe-versions.yaml.

To deploy an individual package, run the workflow zowe-cli-deploy-component.yaml. Specify the package name without the scope (e.g., `core-for-zowe-sdk`) and a space-separated list of tags to publish (e.g., `latest next`).

When a Zowe release has been staged but is not yet GA, it is expected that the Zowe bundle version hosted on zowe.org and the version defined in zowe-versions.yaml@master will differ. During this period, the workflow runs in "staging mode" which means that:
* Packages with "next" tag are not published if their date is newer than the snapshot date defined in zowe-versions.yaml
* Packages with other tags are not published if their version is newer than the one defined in zowe-versions.yaml
  * For the "latest" tag, it is assumed that the 1st tag listed for a package in zowe-versions.yaml is aliased with @latest

Packages defined in the "extras" section of zowe-versions.yaml will always be published and are unaffected by staging mode since they are not included in Zowe CLI bundles.
