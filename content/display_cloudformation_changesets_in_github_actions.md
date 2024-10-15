+++
title = "Display CloudFormation ChangeSets in GitHub Actions"
date = "2024-10-15"

[taxonomies]
tags = ["github"]

[extra]
author = "George Alton"
+++


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
    changeset = cloudformation.describe_change_set(
        ChangeSetName=arn,
        IncludePropertyValues=True,
    )
    return changeset["Changes"]

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
