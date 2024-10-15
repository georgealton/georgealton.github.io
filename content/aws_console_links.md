+++
title = "Improve Developer Experience in CI with AWS Console Links"
date = "2024-10-15"

[taxonomies]
tags = ["github", "aws"]

[extra]
author = "George Alton"
+++

A few months ago AWS Identity Centre relesed a feature to generate links to AWS
Console Pages in member Accounts. 

Using this feature in your CI workflow really improves the Developer
Experience, being able to go straight from a Deployment Error in GitHub to the
AWS Console can be a great way to streamline debugging the issue.

Their isn't an API to generate these these so you'll have to start constructing
them yourself.

The format for the url

```
/#/console?account_id=$AWS_ACCOUNT_ID&role_name=$ROLE_NAME&destination=$CONSOLE_URL
```

The url parameter `role_name` is optional, if you don't specify it then the
person following the link will be asked to choose a Role when entering the
Account.


```
https://[your_subdomain].awsapps.com/start/#/console?account_id=[account_ID]&role_name=[permission_set_name]&destination=[destination_URL]
```

I've found using `jq` to be a reliable way to URL Encode the Parameters we need
to append.

```bash
urlencode () { jq -Rr @uri <<< $1 ;}
```

A useful places to start an investigation from is CloudTrail.

```bash
CLOUDTRAIL_CONSOLE_URL="https://${AWS_REGION}.console.aws.amazon.com/cloudtrailv2/home?region=${AWS_REGION}#/events?"
```

If we just use a link to the CloudTrail Console then we've helped people get to
the right destination but there's more we can do filter down the noise.

When we're assuming Roles from CI it's a great idea to build a Role Session
Name that identifies this CI Run, you can then correlate actions taken.

In Github Actions we can take some information from the Github Context to build
Role Session Name

AWS constrains the maximum length of a `ROLE_SESSION_NAME` to 64 characters. So
we must truncate our length here, we chop off the end of our SHA. This should 
be acceptable, and provide enough fidelity.


`github_role_session_name.sh`
```bash
SHA=${PR_HEAD_SHA:-${GITHUB_SHA}}
ROLE_SESSION_NAME="${GITHUB_RUN_ID}.${GITHUB_RUN_ATTEMPT}+${SHA}"
echo "value=${ROLE_SESSION_NAME:0:64}" >> "${GITHUB_OUTPUT}"
```

```yaml
- name: Role Session Name
  id: role-session-name
  env:
    GH_TOKEN: ${{ inputs.github-token }}
    PR_HEAD_SHA: ${{ github.event.pull_request.head.sha }}
  shell: bash
  run: github_role_session_name.sh
```

to scope CloudTrail Events


```bash
CLOUDTRAIL_CONSOLE_URL="https://${AWS_REGION}.console.aws.amazon.com/cloudtrailv2/home?region=${AWS_REGION}#/events?Username=${ROLE_SESSION_NAME}"
```

```bash
AWS_IDENTITY_CENTRE_SUBDOMAIN="YOUR_SUBDOMAIN"
AWS_IDENTITY_CENTRE_BASE_URL="https://${SUBDOMAIN}.awsapps.com/start/#/console?"
CONSOLE_URL=$(urlencode "${CLOUDTRAIL_CONSOLE_URL}")
LINK_URL="${AWS_IDENTITY_CENTRE_BASE_URL}&account_id=${AWS_ACCOUNT_ID}&destination=${CONSOLE_URL}"
```

This small improvement can have a big impact on how integrated your developers
feel about AWS and Github Actions.

An of course this technique can be expanded further by generate links to other
Resources some ideas: 

- S3 Buckets
- CloudWatch Log Groups
- ECS Deployments
- CloudFormation Resources
