This is a multi-stage CI/CD pipeline triggered on pushes to main or feature/* branches and manually (workflow_dispatch), composed of:

CI Phase
Jobs:

unit-testing

Uses mongodb container service.

Tests on Node.js 18 and 20.

Caches and installs node_modules using a composite action.

Outputs Mocha test-results.xml.

code-coverage

Uses node:20 Docker container and MongoDB service.

Skips setup-node because the container already includes Node.js.

Runs npm run coverage and archives output in coverage/.


Artifact Handling
reports-s3

Downloads both unit test and coverage artifacts.

Merges and uploads them to an AWS S3 bucket.


Containerization
docker

Builds the Docker image.

Tests the image locally with wget on 127.0.0.1:3000/live.

Pushes to both DockerHub and GHCR.


CD Phase
✅ For feature branches:
dev-deploy

Calls the reusable workflow with:

Environment: development

k8s-manifest-dir: kubernetes/development/

dev-integration-testing

Verifies https://$URL/live using curl and jq.

✅ For main branch:
prod-deploy

Calls the reusable workflow with:

Environment: production

k8s-manifest-dir: kubernetes/production/

prod-integration-testing

Similar curl test against the production app.


Notifications
slack-notification

Posts a message to Slack via webhook after both test paths.

Continues on error, so it won’t block the workflow.


Reusable Workflow: reuse-deployment.yml
Triggered by workflow_call, it:

Accepts:

MongoDB URI

Kubernetes manifest directory

Environment name

Kubeconfig secret

MongoDB password

Installs and sets up kubectl, sets the kubeconfig.

Ensures the target Kubernetes namespace exists.

Extracts the Ingress IP for _{{_INGRESS_IP_}}_ substitution.

Uses replace-tokens to template the manifests.

Creates MongoDB secret in the specified namespace.

Applies the manifest files.

Exposes the final ingress URL back to the caller (application-url).
