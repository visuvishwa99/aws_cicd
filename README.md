# AWS CI/CD practice: EdTech event delivery

This lab deploys an SNS topic connected to an SQS queue, the first link in the
event pipeline described by the existing EdTech Data Platform repository. That
project's event generator emits student activity events with `version`,
`changeType`, `eventInfo`, and `payload` fields. The deployment smoke test
publishes an `ActivityCompleted` event in that shape. SQS holds the event until
a worker is ready to consume it.

```text
Change the template -> pull request -> CI: lint and preview
                                  -> merge to main -> CD: deploy and smoke test

EdTech app -> SNS topic -> SQS queue -> EventBridge Pipes -> Firehose -> S3
```

**CI (continuous integration)** checks a proposed change before it is merged.
Here, `cfn-lint` checks both CloudFormation templates. The PR job also asks AWS
to validate the demo template and, once a demo stack exists, creates a temporary
CloudFormation *change set* showing what would change. It never executes that
change set.

**CD (continuous deployment in this lab)** applies a merged change. Here, a
push to `main` deploys the demo stack and then publishes an `ActivityCompleted`
event to SNS, receives it from SQS, and deletes it. If you configure a required
environment reviewer, the deployment waits for that approval; this is a
continuous delivery style. A green deployment means the wiring actually
worked, not just that the template was syntactically valid.

## What is in this folder

| File | Job |
| --- | --- |
| `infra/demo/sns-sqs-demo.yml` | Declares the topic, queue, queue policy, subscription, and output values. |
| `infra/demo/bootstrap-github-roles.yml` | One-time setup of the GitHub OIDC provider and three IAM roles. |
| `.github/workflows/demo-stack.yml` | GitHub Actions CI and CD jobs. |

The **plan role** can validate and preview this stack. The **deploy role** can
apply it and perform the smoke test. The **CloudFormation execution role** has
permissions to create the demo SNS/SQS resources. GitHub gets short-lived AWS
credentials through OIDC; no AWS access key is stored in GitHub.

## Prerequisites

- An AWS account where you are allowed to create CloudFormation, IAM, SNS, and
  SQS resources, plus AWS CLI v2 configured locally.
- A GitHub repository containing the **contents of this `demo_cicd` folder at
  the repository root**. The workflow expects `infra/demo/...` paths relative
  to that root. This folder is now an initialized local Git repository; it has
  not yet been pushed to GitHub.
- A `main` branch with GitHub Actions enabled. Use a disposable AWS account or
  a region where this practice stack will not collide with existing resources.
  AWS resources may incur charges; delete the stacks when finished.

Run these commands in PowerShell from the repository root (the directory that
contains `.github` and `infra`):

```powershell
$Region = 'us-east-1' # choose your region and use it consistently
aws sts get-caller-identity
aws cloudformation validate-template --template-body file://infra/demo/bootstrap-github-roles.yml --region $Region
aws cloudformation validate-template --template-body file://infra/demo/sns-sqs-demo.yml --region $Region
```

## 1. Bootstrap AWS access once

Check whether your account already has GitHub's OIDC provider. IAM providers
are account-wide, so create only one for `token.actions.githubusercontent.com`:

```powershell
aws iam list-open-id-connect-providers --output table
```

If it exists, pass its ARN as an additional parameter in the command below:
`ExistingGitHubOidcProviderArn=arn:aws:iam::ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com`.
Otherwise, the bootstrap stack creates it. Replace `OWNER/REPO` with the exact
GitHub repository name.

```powershell
aws cloudformation deploy `
  --stack-name learn-sns-sqs-bootstrap `
  --template-file infra/demo/bootstrap-github-roles.yml `
  --parameter-overrides GitHubRepository=OWNER/REPO `
  --capabilities CAPABILITY_IAM `
  --region $Region

aws cloudformation describe-stacks `
  --stack-name learn-sns-sqs-bootstrap `
  --query 'Stacks[0].Outputs' `
  --output table `
  --region $Region
```

This is intentionally a manual bootstrap: the pipeline cannot grant itself
the IAM permissions needed to run. If the OIDC provider already exists, add
`ExistingGitHubOidcProviderArn=...` to `--parameter-overrides` above.

## 2. Configure GitHub

In **Settings → Secrets and variables → Actions → Variables**, create:

| Variable | Value |
| --- | --- |
| `AWS_REGION` | The region used for bootstrap. |
| `DEMO_PLAN_ROLE_ARN` | `PlanRoleArn` output. |
| `DEMO_DEPLOY_ROLE_ARN` | `DeployRoleArn` output. |
| `DEMO_CFN_EXEC_ROLE_ARN` | `ExecRoleArn` output. |

Create a GitHub environment named **`demo-deploy`**. Set its deployment branch
rule to **`main`**. You may add a required reviewer if you want a human pause
between CI and deployment. The IAM deploy role trusts this environment name,
so the spelling must match. The plan job uses the PR identity. Use a PR branch
in the same repository for the AWS preview; fork PRs run lint only.

## 3. Run the lab

1. Open a PR that changes `infra/demo/sns-sqs-demo.yml`. For a harmless first
   edit, change the template `Description`. Watch **lint**, then **plan** in
   the Actions run. On the first PR, the plan says the stack does not exist yet;
   creating a change set for a new stack would leave a placeholder stack.
2. Merge the PR to `main`. Watch **lint**, **deploy**, and the **smoke test**.
   In AWS CloudFormation, inspect `learn-sns-sqs` and its **Resources** and
   **Outputs** tabs. No deployment happens from a PR.
3. Create a second PR. Change `MessageRetentionPeriod` in the queue from
   `345600` (4 days) to `172800` (2 days). The PR change set should show a
   queue **Modify** action. Read the `Replacement` column before merging.
4. Merge and verify the smoke test passes again. Check the queue in the SQS
   console or inspect the stack events to connect each pipeline step to an
   AWS action.

You can also run **Actions → SNS/SQS demo stack → Run workflow** on `main` to
redeploy without changing the template. An empty change set is a valid no-op.

## What to inspect when something fails

| Failure | First place to look |
| --- | --- |
| `cfn-lint` | Template path and resource/property name in the lint output. |
| `AssumeRoleWithWebIdentity` | GitHub repo name, environment name, branch rule, OIDC provider, and role ARN. |
| Change set | The plan job's `StatusReason` and CloudFormation change sets. |
| Deploy | CloudFormation **Events** for the first failed resource. |
| Smoke test | Stack outputs, SNS subscription, SQS queue policy, and message delivery. |

The first AWS preview deliberately skips a change set until the demo stack
exists. If an older attempt left `learn-sns-sqs` in `REVIEW_IN_PROGRESS`, remove
that empty stack in CloudFormation before retrying. Do not remove a running
stack just to fix a preview.

## Clean up

After the exercise, delete the demo stack first, then the bootstrap stack:

```powershell
aws cloudformation delete-stack --stack-name learn-sns-sqs --region $Region
aws cloudformation wait stack-delete-complete --stack-name learn-sns-sqs --region $Region
aws cloudformation delete-stack --stack-name learn-sns-sqs-bootstrap --region $Region
aws cloudformation wait stack-delete-complete --stack-name learn-sns-sqs-bootstrap --region $Region
```

The bootstrap stack deletes the GitHub OIDC provider only if it created that
provider. If you want to keep practising, leave the bootstrap stack in place
and delete only the demo stack.

## Next Lexia exercises

Once this flow is familiar, add a dead-letter queue for failed messages, then
a small Lambda consumer that writes a processing result. Keep the same rhythm:
template change → PR checks and preview → merge → deployment → end-to-end
smoke test. Each new service makes the CI/CD responsibilities clearer.

## Reference docs

- [CloudFormation change sets](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-updating-stacks-changesets.html)
- [GitHub OIDC claims](https://docs.github.com/en/actions/reference/security/oidc)
- [AWS credentials action](https://github.com/aws-actions/configure-aws-credentials)
