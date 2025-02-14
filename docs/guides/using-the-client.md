---
description: Learn how to leverage the `PrefectClient` to interact with the API.
tags:
    - client
    - API
    - filters
    - orchestration
---

# Using the Prefect Orchestration Client
## Overview
In the [API reference for the `PrefectClient`](/api-ref/prefect/client/orchestration/), you can find a bunch of useful client methods that make it simpler to do things like:

- [reschedule late flow runs](#rescheduling-late-flow-runs)
- [get the last `N` completed flow runs from my workspace](#get-the-last-n-completed-flow-runs-from-my-workspace)

The `PrefectClient` is an async context manager, so you can use it like this:
```python hl_lines="3"
from prefect import get_client

async with get_client() as client:
    response = await client.hello()
    print(response.json()) # 👋
```


## Examples

### Rescheduling late flow runs
Sometimes, you may need to bulk reschedule flow runs that are late - for example, if you've accidentally scheduled many flow runs of a deployment to an inactive work pool.

To do this, we can delete late flow runs and create new ones in a `Scheduled` state with a delay.

This example reschedules the last 3 late flow runs of a deployment named `healthcheck-storage-test` to run 6 hours later than their original expected start time. It also deletes any remaining late flow runs of that deployment.

```python
import asyncio
from datetime import datetime, timedelta, timezone
from typing import Optional

from prefect import get_client
from prefect.client.schemas.filters import (
    DeploymentFilter, FlowRunFilter
)
from prefect.client.schemas.objects import FlowRun
from prefect.client.schemas.sorting import FlowRunSort
from prefect.states import Scheduled

async def reschedule_late_flow_runs(
    deployment_name: str,
    delay: timedelta,
    most_recent_n: int,
    delete_remaining: bool = True,
    states: Optional[list[str]] = None
) -> list[FlowRun]:
    if not states:
        states = ["Late"]

    async with get_client() as client:
        flow_runs = await client.read_flow_runs(
            flow_run_filter=FlowRunFilter(
                state=dict(name=dict(any_=states)),
                expected_start_time=dict(
                    before_=datetime.now(timezone.utc)
                ),
            ),
            deployment_filter=DeploymentFilter(
                name={'like_': deployment_name}
            ),
            sort=FlowRunSort.START_TIME_DESC,
            limit=most_recent_n if not delete_remaining else None
        )

        if not flow_runs:
            print(f"No flow runs found in states: {states!r}")
            return []
        
        rescheduled_flow_runs = []
        for i, run in enumerate(flow_runs):
            await client.delete_flow_run(flow_run_id=run.id)
            if i < most_recent_n:
                new_run = await client.create_flow_run_from_deployment(
                    deployment_id=run.deployment_id,
                    state=Scheduled(
                        scheduled_time=run.expected_start_time + delay
                    ),
                )
                rescheduled_flow_runs.append(new_run)
            
        return rescheduled_flow_runs

if __name__ == "__main__":
    rescheduled_flow_runs = asyncio.run(
        reschedule_late_flow_runs(
            deployment_name="healthcheck-storage-test",
            delay=timedelta(hours=6),
            most_recent_n=3,
        )
    )
    
    print(f"Rescheduled {len(rescheduled_flow_runs)} flow runs")
        
    assert all(
        run.state.is_scheduled() for run in rescheduled_flow_runs
    )
    assert all(
        run.expected_start_time > datetime.now(timezone.utc)
        for run in rescheduled_flow_runs
    )
```

### Get the last `N` completed flow runs from my workspace
To get the last `N` completed flow runs from our workspace, we can make use of `read_flow_runs` and `prefect.client.schemas`.

This example gets the last three completed flow runs from our workspace:
```python
import asyncio
from typing import Optional

from prefect import get_client
from prefect.client.schemas.filters import FlowRunFilter
from prefect.client.schemas.objects import FlowRun
from prefect.client.schemas.sorting import FlowRunSort

async def get_most_recent_flow_runs(
    n: int = 3,
    states: Optional[list[str]] = None
) -> list[FlowRun]:
    if not states:
        states = ["COMPLETED"]
    
    async with get_client() as client:
        return await client.read_flow_runs(
            flow_run_filter=FlowRunFilter(
                state={'type': {'any_': states}}
            ),
            sort=FlowRunSort.END_TIME_DESC,
            limit=n,
        )

if __name__ == "__main__":
    last_3_flow_runs: list[FlowRun] = asyncio.run(
        get_most_recent_flow_runs()
    )
    print(last_3_flow_runs)
    
    assert all(
        run.state.is_completed() for run in last_3_flow_runs
    )
    assert (
        end_times := [run.end_time for run in last_3_flow_runs]
    ) == sorted(end_times, reverse=True)
```

Instead of the last three from the whole workspace, you could also use the `DeploymentFilter` like the previous example to get the last three completed flow runs of a specific deployment.

!!! tip "There are other ways to filter objects like flow runs"
    See [`the filters API reference`](/api-ref/prefect/client/schemas/#prefect.client.schemas.filters) for more ways to filter flow runs and other objects in your Prefect ecosystem.
