# 6. Interacting with the Server

With the Helloworld A2A server running, let's send some requests to it. The SDK includes a client (`A2AClient`) that simplifies these interactions.

## The Helloworld Test Client

The `samples/helloworld/test_client.py` script demonstrates how to:

1. Fetch the Agent Card from the server.
2. Create an `A2AClient` instance.
3. Send both non-streaming (`message/send`) and streaming (`message/stream`) requests.

Open a **new terminal window**, activate your virtual environment, and navigate to the `a2a-samples` directory.

Activate virtual environment (Be sure to do this in the same directory where you created the virtual environment):

=== "Mac/Linux"

    ```sh
    source .venv/bin/activate
    ```

=== "Windows"

    ```powershell
    .venv\Scripts\activate
    ```

Run the test client:

```bash
# from the a2a-samples directory
python samples/helloworld/test_client.py
```

## Understanding the Client Code

Let's look at key parts of `samples/helloworld/test_client.py`:

1. **Fetching the Agent Card & Initializing the Client**:

    ```python { .no-copy }
    # samples/helloworld/test_client.py
    async with httpx.AsyncClient() as httpx_client:
        client = await A2AClient.get_client_from_agent_card_url(
            httpx_client, 'http://localhost:9999'
        )
    ```

    The `A2AClient.get_client_from_agent_card_url` class method is a convenience. It first fetches the `AgentCard` from the server's `/.well-known/agent.json` endpoint (based on the provided base URL) and then initializes the client with it.

2. **Sending a Non-Streaming Message (`send_message`)**:

    ```python { .no-copy }
    # samples/helloworld/test_client.py
    from a2a.types import (
        MessageSendParams,
        SendMessageRequest,
        SendStreamingMessageRequest,
    )
    # ...
    send_message_payload: dict[str, Any] = {
        'message': {
            'role': 'user',
            'parts': [{'type': 'text', 'text': 'how much is 10 USD in INR?'}], # Content doesn't matter for Helloworld
            'messageId': uuid4().hex,
        },
    }
    request = SendMessageRequest(
        params=MessageSendParams(**send_message_payload)
    )

    response = await client.send_message(request)
    print(response.model_dump(mode='json', exclude_none=True))
    ```

    - The `send_message_payload` constructs the data for `MessageSendParams`.
    - This is wrapped in a `SendMessageRequest`.
    - It includes a `message` object with the `role` set to "user" and the content in `parts`.
    - The Helloworld agent's `execute` method will enqueue a single "Hello World" message. The `DefaultRequestHandler` will retrieve this and send it as the response.
    - The `response` will be a `SendMessageResponse` object, which contains either a `SendMessageSuccessResponse` (with the agent's `Message` as the result) or a `JSONRPCErrorResponse`.

3. **Handling Task IDs (Illustrative Note for Helloworld)**:
    The Helloworld client (`samples/helloworld/test_client.py`) doesn't attempt `get_task` or `cancel_task` directly because the simple Helloworld agent's `execute` method, when called via `message/send`, results in the `DefaultRequestHandler` returning a direct `Message` response rather than a `Task` object. More complex agents that explicitly manage tasks (like the LangGraph example) would return a `Task` object from `message/send`, and its `id` could then be used for `get_task` or `cancel_task`.

4. **Sending a Streaming Message (`send_message_streaming`)**:

    ```python { .no-copy }
    # samples/helloworld/test_client.py
    streaming_request = SendStreamingMessageRequest(
        params=MessageSendParams(**send_message_payload) # Same payload can be used
    )

    stream_response = client.send_message_streaming(streaming_request)
    async for chunk in stream_response:
        print(chunk.model_dump(mode='json', exclude_none=True))
    ```

    - This method calls the agent's `message/stream` endpoint. The `DefaultRequestHandler` will invoke the `HelloWorldAgentExecutor.execute` method.
    - The `execute` method enqueues one "Hello World" message, and then the event queue is closed.
    - The client will receive this single message as one `SendStreamingMessageResponse` event, and then the stream will terminate.
    - The `stream_response` is an `AsyncGenerator`.

## Expected Output

When you run `test_client.py`, you'll see JSON outputs for:

- The non-streaming response (a single "Hello World" message).
- The streaming response (a single "Hello World" message as one chunk, after which the stream ends).

The `id` fields in the output will vary with each run.

```console { .no-copy }
// Non-streaming response
{"jsonrpc":"2.0","id":"xxxxxxxx","result":{"type":"message","role":"agent","parts":[{"type":"text","text":"Hello World"}],"messageId":"yyyyyyyy"}}
// Streaming response (one chunk)
{"jsonrpc":"2.0","id":"zzzzzzzz","result":{"type":"message","role":"agent","parts":[{"type":"text","text":"Hello World"}],"messageId":"wwwwwwww","final":true}}
```

*(Actual IDs like `xxxxxxxx`, `yyyyyyyy`, `zzzzzzzz`, `wwwwwwww` will be different UUIDs/request IDs)*

This confirms your server is correctly handling basic A2A interactions with the updated SDK structure!

Now you can shut down the server by typing Ctrl+C in the terminal window where `__main__.py` is running.
