# Proposal: Check Hook Implementation

Author(s): @nikkiattea @palourde

Last updated: 11/27/17

Discussion at https://github.com/sensu/sensu-go/issues/495

## Abstract

Modify the check hook resource to allow reusability amongst checks, descriptive name uniqueness, and non-configuration management friendliness.

## Background

Check hooks are not reusable in 1.x, because they lack unique and descriptive identifiers and are only stored within the scope of a check configuration. 1.x Checks are also limited to one hook per severity/response code.

## Proposal

Reform the check hook resource in such a way that it contains a name descriptive to the hook action/command and is consistent with 2.0 resource implementations, ultimately allowing reusability of hooks across different checks for potentially different check responses.

1.x Hook
```
{
    “checks”: {
        “nginx_process”: {
            “command”: “check-process.rb -p nginx”,
            “subscribers”: [“proxy”],
            “interval”: 30,
            “hooks”: {
                “critical”: {
                    “command”: “sudo /etc/init.d/nginx start”,
                    “timeout”: 30,
                },
                “non-zero”: {
                    “command”: “ps aux”,
                    “timeout”: 10,
                },
            }
        }
    }
}
```

2.0 Proposed Hook
```
{
    “checks”: {
        “nginx_process”: {
            “command”: “check-process.rb -p nginx”,
            “subscribers”: [“proxy”],
            “interval”: 30,
            "check_hooks": [
            {
                “critical”: [“nginx-start”, “another-critical-hook??”],
            },
            {
                "non-zero”: [“ps-aux”],
            }],
        }
    }
}

{
    “hooks”: {
        “nginx-start”: {
            “command”: “sudo /etc/init.d/nginx start”,
            “timeout”: 30,
        },
        “ps-aux” : {
            “command”: “ps aux”,
            “timeout”: 10,
        }
    }
}
```

## Rationale

Advantages: In the example above, the check “nginx_process” can contain multiple hooks for a “critical” response. The hook “ps-aux” can appear in both the check “nginx_process” as a “non-zero” response and in another arbitrary check under a different response, it is not bound to a single response.
Disadvantages: A hook can be created independently (outside the check configuration scope) and does not require a check association. However, the hook will not execute without an association to a check.
Alternate approach: Hooks are stored within the check configuration scope (limits reusability) or hooks are stored in their own store but require multiple identifiers: hook name + check name = unique identifier (messy).

## Compatibility

I don’t believe this proposal to be a completely breaking change. We would still fulfill 1.x parity by implementing check hooks, and they would be implemented similarly to our other 2.x resources. However, this proposal makes migration from 1.x more difficult. We would need another mean for importing or accept that the resource cannot be ported automatically.

## Implementation

This will be implemented as part of https://github.com/sensu/sensu-go/issues/495 which should be done this week. This proposal will likely have a faster implementation than any alternative.

## Open issues (if applicable)

https://github.com/sensu/sensu-go/issues/495
https://github.com/sensu/sensu-go/issues/643
https://github.com/sensu/sensu-go/issues/645
