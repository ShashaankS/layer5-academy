---
docType: "Chapter"
id: "view-application-logs"
description: "Use Meshery's interactive terminal to view logs of applications"
lectures: 4
weight: 6
title: "View Application Logs"
---

In this chapter, we will explore how Dapr utilizes its [API constructs](https://docs.dapr.io/concepts/building-blocks-concept/) to facilitate communication and manage the state of application data within this architecture by observing the container logs.

### Python Application Logs

The Python application code above generates messages that contain data with an **orderId** that increments once per second.

Here's a snippet from the Python script in **app.js**:

```python
dapr_port = os.getenv("DAPR_HTTP_PORT", 3500)
dapr_url = "http://localhost:{}/neworder".format(dapr_port)

n = 0
while True:
    n += 1
    message = {"data": {"orderId": n}}

    try:
        response = requests.post(dapr_url, json=message)
    except Exception as e:
        print(e)

    time.sleep(1)
```

The Dapr sidecar for the Python application sends a POST request to the Dapr sidecar of the Node.js application using the **Dapr service invocation API**. Here's a breakdown of what happens:

1. The Python app sends a POST request to its sidecar at http://localhost:3500/v1.0/invoke/nodeapp/method/neworder. _Note: http://localhost:3500 is the default listening port for Dapr_.
2. The sidecar for the Python app invokes the nodeapp service through its own Dapr sidecar.

The Python app does not need to know the exact address or port of the Node.js service, it simply makes a request to its sidecar, which handles the routing.

**Steps to Stream Python Application Logs**:

1. Click on the **python-app** pod.
2. On the right sidebar, click on **Action**.
3. Click on **Stream Container logs**.

![stream](stream.png)

The logs show the daprd container logs with a POST request made to **/neworder** endpoint.

### Node.js Application Logs

Follow the same steps above to get the logs for the Node.js application. The logs show API calls made to the state store for persisting order data.

Here's what can be observed from the logs:

1. The Dapr sidecar of the Node.js application listens on port 3500 for incoming HTTP requests.
2. The sidecar receives the invocation request sent from the sidecar of the Python app and routes it to the **/neworder** endpoint in the Node.js application.
3. The sidecar makes a POST request to the state store endpoint (/v1.0/state/statestore) to persist the state information in Redis. This endpoint is part of the **Dapr state management API**  and is mapped to the configured state store component.

By analyzing these logs, we gained a deeper understanding of how Dapr's APIs such as the state management API and service invocation API work together to enable service-service communication and efficient management of application state.
