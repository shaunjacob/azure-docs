---
title: Scheduled Events for Linux VMs in Azure 
description: Schedule events by using Azure Metadata Service for your Linux virtual machines.
author: EricRadzikowskiMSFT
ms.service: virtual-machines
ms.subservice: scheduled-events
ms.collection: linux
ms.topic: how-to
ms.workload: infrastructure-services
ms.date: 06/01/2020
ms.author: ericrad
ms.reviewer: mimckitt

---

# Azure Metadata Service: Scheduled Events for Linux VMs

**Applies to:** :heavy_check_mark: Linux VMs :heavy_check_mark: Flexible scale sets :heavy_check_mark: Uniform scale sets 

Scheduled Events is an Azure Metadata Service that gives your application time to prepare for virtual machine (VM) maintenance. It provides information about upcoming maintenance events (for example, reboot) so that your application can prepare for them and limit disruption. It's available for all Azure Virtual Machines types, including PaaS and IaaS on both Windows and Linux. 

For information about Scheduled Events on Windows, see [Scheduled Events for Windows VMs](../windows/scheduled-events.md).

> [!Note] 
> Scheduled Events is generally available in all Azure Regions. See [Version and Region Availability](#version-and-region-availability) for latest release information.

## Why use Scheduled Events?

Many applications can benefit from time to prepare for VM maintenance. The time can be used to perform application-specific tasks that improve availability, reliability, and serviceability, including: 

- Checkpoint and restore.
- Connection draining.
- Primary replica failover.
- Removal from a load balancer pool.
- Event logging.
- Graceful shutdown.

With Scheduled Events, your application can discover when maintenance will occur and trigger tasks to limit its impact.  

Scheduled Events provides events in the following use cases:

- [Platform initiated maintenance](../maintenance-and-updates.md?bc=/azure/virtual-machines/linux/breadcrumb/toc.json&toc=/azure/virtual-machines/linux/toc.json) (for example, VM reboot, live migration or memory preserving updates for host)
- Virtual machine is running on [degraded host hardware](https://azure.microsoft.com/blog/find-out-when-your-virtual-machine-hardware-is-degraded-with-scheduled-events) that is predicted to fail soon
- User-initiated maintenance (for example, a user restarts or redeploys a VM)
- [Spot VM](../spot-vms.md) and [Spot scale set](../../virtual-machine-scale-sets/use-spot.md) instance evictions.

## The Basics  

  Metadata Service exposes information about running VMs by using a REST endpoint that's accessible from within the VM. The information is available via a nonroutable IP so that it's not exposed outside the VM.

### Scope
Scheduled events are delivered to:

- Standalone Virtual Machines.
- All the VMs in a cloud service.
- All the VMs in an availability set.
- All the VMs in an availability zone. 
- All the VMs in a scale set placement group. 

> [!NOTE]
> Specific to VMs in an availability zone, the scheduled events go to single VMs in a zone.
> For example, if you have 100 VMs in a availability set and there is an update to one of them, the scheduled event will go to all 100, whereas if there are 100 single VMs in a zone, then event will only go to the VM which is getting impacted.

As a result, check the `Resources` field in the event to identify which VMs are affected.

### Endpoint Discovery
For VNET enabled VMs, Metadata Service is available from a static nonroutable IP, `169.254.169.254`. The full endpoint for the latest version of Scheduled Events is: 

 > `http://169.254.169.254/metadata/scheduledevents?api-version=2020-07-01`

If the VM is not created within a Virtual Network, the default cases for cloud services and classic VMs, additional logic is required to discover the IP address to use. 
To learn how to [discover the host endpoint](https://github.com/azure-samples/virtual-machines-python-scheduled-events-discover-endpoint-for-non-vnet-vm), see this sample.

### Version and Region Availability
The Scheduled Events service is versioned. Versions are mandatory; the current version is `2020-07-01`.

| Version | Release Type | Regions | Release Notes | 
| - | - | - | - | 
| 2020-07-01 | General Availability | All | <li> Added support for Event Duration |
| 2019-08-01 | General Availability | All | <li> Added support for Event Source |
| 2019-04-01 | General Availability | All | <li> Added support for Event Description |
| 2019-01-01 | General Availability | All | <li> Added support for virtual machine scale sets EventType 'Terminate' |
| 2017-11-01 | General Availability | All | <li> Added support for Spot VM eviction EventType 'Preempt'<br> | 
| 2017-08-01 | General Availability | All | <li> Removed prepended underscore from resource names for IaaS VMs<br><li>Metadata header requirement enforced for all requests | 
| 2017-03-01 | Preview | All | <li>Initial release |


> [!NOTE] 
> Previous preview releases of Scheduled Events supported {latest} as the api-version. This format is no longer supported and will be deprecated in the future.

### Enabling and Disabling Scheduled Events
Scheduled Events is enabled for your service the first time you make a request for events. You should expect a delayed response in your first call of up to two minutes.

Scheduled Events is disabled for your service if it does not make a request for 24 hours.

### User-initiated Maintenance
User-initiated VM maintenance via the Azure portal, API, CLI, or PowerShell results in a scheduled event. You then can test the maintenance preparation logic in your application, and your application can prepare for user-initiated maintenance.

If you restart a VM, an event with the type `Reboot` is scheduled. If you redeploy a VM, an event with the type `Redeploy` is scheduled.

## Use the API

### Headers
When you query Metadata Service, you must provide the header `Metadata:true` to ensure the request wasn't unintentionally redirected. The `Metadata:true` header is required for all scheduled events requests. Failure to include the header in the request results in a "Bad Request" response from Metadata Service.

### Query for events
You can query for scheduled events by making the following call:

#### Bash
```
curl -H Metadata:true http://169.254.169.254/metadata/scheduledevents?api-version=2020-07-01
```

A response contains an array of scheduled events. An empty array means that currently no events are scheduled.
In the case where there are scheduled events, the response contains an array of events. 
```
{
    "DocumentIncarnation": {IncarnationID},
    "Events": [
        {
            "EventId": {eventID},
            "EventType": "Reboot" | "Redeploy" | "Freeze" | "Preempt" | "Terminate",
            "ResourceType": "VirtualMachine",
            "Resources": [{resourceName}],
            "EventStatus": "Scheduled" | "Started",
            "NotBefore": {timeInUTC},       
            "Description": {eventDescription},
            "EventSource" : "Platform" | "User",
            "DurationInSeconds" : {timeInSeconds},
        }
    ]
}
```

### Event Properties
|Property  |  Description |
| - | - |
| EventId | Globally unique identifier for this event. <br><br> Example: <br><ul><li>602d9444-d2cd-49c7-8624-8643e7171297  |
| EventType | Impact this event causes. <br><br> Values: <br><ul><li> `Freeze`: The Virtual Machine is scheduled to pause for a few seconds. CPU and network connectivity may be suspended, but there is no impact on memory or open files.<li>`Reboot`: The Virtual Machine is scheduled for reboot (non-persistent memory is lost). This event is made available on a best effort basis <li>`Redeploy`: The Virtual Machine is scheduled to move to another node (ephemeral disks are lost). <li>`Preempt`: The Spot Virtual Machine is being deleted (ephemeral disks are lost). <li> `Terminate`: The virtual machine is scheduled to be deleted. |
| ResourceType | Type of resource this event affects. <br><br> Values: <ul><li>`VirtualMachine`|
| Resources| List of resources this event affects. The list is guaranteed to contain machines from at most one [update domain](../availability.md), but it might not contain all machines in the UD. <br><br> Example: <br><ul><li> ["FrontEnd_IN_0", "BackEnd_IN_0"] |
| EventStatus | Status of this event. <br><br> Values: <ul><li>`Scheduled`: This event is scheduled to start after the time specified in the `NotBefore` property.<li>`Started`: This event has started.</ul> No `Completed` or similar status is ever provided. The event is no longer returned when the event is finished.
| NotBefore| Time after which this event can start. <br><br> Example: <br><ul><li> Mon, 19 Sep 2016 18:29:47 GMT  |
| Description | Description of this event. <br><br> Example: <br><ul><li> Host server is undergoing maintenance. |
| EventSource | Initiator of the event. <br><br> Example: <br><ul><li> `Platform`: This event is initiated by platform. <li>`User`: This event is initiated by user. |
| DurationInSeconds | The expected duration of the interruption caused by the event.  <br><br> Example: <br><ul><li> `9`: The interruption caused by the event will last for 9 seconds. <li>`-1`: The default value used if the impact duration is either unknown or not applicable. |

### Event Scheduling
Each event is scheduled a minimum amount of time in the future based on the event type. This time is reflected in an event's `NotBefore` property. 

|EventType  | Minimum notice |
| - | - |
| Freeze| 15 minutes |
| Reboot | 15 minutes |
| Redeploy | 10 minutes |
| Preempt | 30 seconds |
| Terminate | [User Configurable](../../virtual-machine-scale-sets/virtual-machine-scale-sets-terminate-notification.md#enable-terminate-notifications): 5 to 15 minutes |

> [!NOTE] 
> In some cases, Azure is able to predict host failure due to degraded hardware and will attempt to mitigate disruption to your service by scheduling a migration. Affected virtual machines will receive a scheduled event with a `NotBefore` that is typically a few days in the future. The actual time varies depending on the predicted failure risk assessment. Azure tries to give 7 days' advance notice when possible, but the actual time varies and might be smaller if the prediction is that there is a high chance of the hardware failing imminently. To minimize risk to your service in case the hardware fails before the system-initiated migration, we recommend that you self-redeploy your virtual machine as soon as possible.

### Polling frequency

You can poll the endpoint for updates as frequently or infrequently as you like. However, the longer the time between requests, the more time you potentially lose to react to an upcoming event. Most events have 5 to 15 minutes of advance notice, although in some cases advance notice might be as little as 30 seconds. To ensure that you have as much time as possible to take mitigating actions, we recommend that you poll the service once per second.

### Start an event 

After you learn of an upcoming event and finish your logic for graceful shutdown, you can approve the outstanding event by making a `POST` call to Metadata Service with `EventId`. This call indicates to Azure that it can shorten the minimum notification time (when possible). 

The following JSON sample is expected in the `POST` request body. The request should contain a list of `StartRequests`. Each `StartRequest` contains `EventId` for the event you want to expedite:
```
{
	"StartRequests" : [
		{
			"EventId": {EventId}
		}
	]
}
```

#### Bash sample
```
curl -H Metadata:true -X POST -d '{"StartRequests": [{"EventId": "f020ba2e-3bc0-4c40-a10b-86575a9eabd5"}]}' http://169.254.169.254/metadata/scheduledevents?api-version=2020-07-01
```

> [!NOTE] 
> Acknowledging an event allows the event to proceed for all `Resources` in the event, not just the VM that acknowledges the event. Therefore, you can choose to elect a leader to coordinate the acknowledgement, which might be as simple as the first machine in the `Resources` field.

## Python sample 

The following sample queries Metadata Service for scheduled events and approves each outstanding event:

```python
#!/usr/bin/python

import json
import socket
import urllib2

metadata_url = "http://169.254.169.254/metadata/scheduledevents?api-version=2020-07-01"
this_host = socket.gethostname()


def get_scheduled_events():
    req = urllib2.Request(metadata_url)
    req.add_header('Metadata', 'true')
    resp = urllib2.urlopen(req)
    data = json.loads(resp.read())
    return data


def handle_scheduled_events(data):
    for evt in data['Events']:
        eventid = evt['EventId']
        status = evt['EventStatus']
        resources = evt['Resources']
        eventtype = evt['EventType']
        resourcetype = evt['ResourceType']
        notbefore = evt['NotBefore'].replace(" ", "_")
	description = evt['Description']
	eventSource = evt['EventSource']
        if this_host in resources:
            print("+ Scheduled Event. This host " + this_host +
                " is scheduled for " + eventtype + 
		" by " + eventSource + 
		" with description " + description +
		" not before " + notbefore)
            # Add logic for handling events here


def main():
    data = get_scheduled_events()
    handle_scheduled_events(data)


if __name__ == '__main__':
    main()
```

## Next steps 
- Review the Scheduled Events code samples in the [Azure Instance Metadata Scheduled Events GitHub repository](https://github.com/Azure-Samples/virtual-machines-scheduled-events-discover-endpoint-for-non-vnet-vm).
- Read more about the APIs that are available in the [Instance Metadata Service](instance-metadata-service.md).
- Learn about [planned maintenance for Linux virtual machines in Azure](../maintenance-and-updates.md?bc=/azure/virtual-machines/linux/breadcrumb/toc.json&toc=/azure/virtual-machines/linux/toc.json).
