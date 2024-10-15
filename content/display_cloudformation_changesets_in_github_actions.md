+++
title = "Display CloudFormation ChangeSets in GitHub Actions"
date = "2024-10-15"

[taxonomies]
tags = ["github", "aws", "cloudformation"]

[extra]
author = "George Alton"
+++

A while ago now CloudFormation released Property Level Diffs __FIND_ARTICLE__
this got me thinking about how we're able to utilise ChangeSets more in
CI Workflows.

Switching context from the Pull Request view in GitHub Actions to going to the
AWS Console is inconvenient. We want to make our reviewers lives as easy as
possible by providing relevant information on the Pull Request itself.

It's no secret that I have a fondness for CloudFormation, but have always found
the granularity that you get from a `terraform plan` useful when you're 
uncertain, or want to provide certainty to others about the effects of a 
infrastructure change.

```python
from difflib import unified_diff
from dataclasses import asdict, dataclass
from os import environ

from boto3 import client
from yaml import safe_load as loads, safe_dump as dumps
from tabulate import tabulate


@dataclass
class ChangeSummary:
    action: str
    physical_resource_id: str
    logical_resource_id: str | None
    resource_type: str
    replacement: bool
    change: str

    @classmethod
    def from_change(cls, change):
        resource_change = change["ResourceChange"]
        action = resource_change["Action"]
        physical_resource_id = resource_change.get("PhysicalResourceId")
        logical_resource_id = resource_change["LogicalResourceId"]
        replacement = resource_change.get("Replacement")
        resource_type = resource_change["ResourceType"]
        before = resource_change.get("BeforeContext", "")
        after = resource_change.get("AfterContext", "")
        change_diff = diff(normalize(before), normalize(after))
        diff_html = f'<pre lang="diff"><code>{change_diff}</code></pre>'

        return cls(
            action=action,
            physical_resource_id=physical_resource_id,
            logical_resource_id=logical_resource_id,
            resource_type=resource_type,
            replacement=replacement,
            change=diff_html,
        )

def normalize(context):
    return dumps(loads(context), indent=2, sort_keys=True)

def summaries(changes):
    yield from (asdict(ChangeSummary.from_change(change)) for change in changes)

def get_changes(arn):
    return cloudformation.describe_change_set(
        ChangeSetName=arn, IncludePropertyValues=True
    )

def main():
    cloudformation = client("cloudformation")
    change_set_arn = environ["CHANGE_SET_ID"]
    changes = get_changes(change_set_arn)
    change_summaries = summaries(changes)
    table = tabulate(change_summaries, headers="keys", tablefmt="unsafehtml")
    print(table)

def diff(before, after):
    return "".join(
        unified_diff(
            before.splitlines(keepends=True),
            after.splitlines(keepends=True),
        )
    )

if __name__ == "__main__":
    main()
```
