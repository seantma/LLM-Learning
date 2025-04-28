# Chapter 6: Agent Execution Loop

In the [previous chapter](05_threadmanager__agentpress__.md), we met the **ThreadManager**, the "project manager" that keeps track of a specific conversation or task, managing messages and coordinating tool use. It's great at handling *one step* of the conversation – like taking the current history, asking the AI for its next move, and maybe running a tool.

But agents often need multiple steps to finish a job. If you ask `suna` to "Write a python script to fetch weather data, save it to a file, and tell me the temperature," that's not just one step! The agent needs to:

1.  Plan the steps (maybe using `todo.md`).
2.  Write the script (using a file tool).
3.  Run the script (using a shell tool).
4.  Read the result (using a file or shell tool).
5.  Tell you the temperature (using a message).

How does the agent keep going from one step to the next until the whole task is done? It needs something to drive this multi-step process.

## What's the Problem? Keeping the Agent Running

Imagine a car engine. The pistons firing (like the [ThreadManager (AgentPress)](05_threadmanager__agentpress__.md) doing one step) are essential, but you need a crankshaft and timing system to keep them firing in the right sequence, over and over, to make the car move forward.

Our agent needs a similar mechanism. We need something that:

*   Repeatedly asks the agent (via the [ThreadManager (AgentPress)](05_threadmanager__agentpress__.md)) "Okay, what's the next step?" based on the current conversation and goals.
*   Handles the response from the agent (text, tool calls).
*   Executes the requested actions (delegating tool calls to the [ThreadManager (AgentPress)](05_threadmanager__agentpress__.md)).
*   Checks if the task is finished or if the agent needs help.
*   Keeps this cycle going until the job is done.

This continuous cycle is the **Agent Execution Loop**.

## Meet the Agent's "Heartbeat": The Execution Loop

The **Agent Execution Loop** is the main operational cycle the agent follows. Think of it as the agent's "heartbeat" or the main `while` loop in a program that keeps everything running.

It's a process that continuously does the following:

1.  **Prepare:** Checks prerequisites like billing status and gathers any temporary context (like the current state of the browser if the agent was just using it).
2.  **Consult Manager:** Calls the [ThreadManager (AgentPress)](05_threadmanager__agentpress__.md)'s `run_thread` method. This is where the agent "thinks" – the ThreadManager gets the history, calls the LLM, and processes the response (including potentially running tools).
3.  **Process Results:** Receives the results (assistant's text, tool outputs) back from the `run_thread` method, often as a stream of information chunks.
4.  **Communicate:** Sends these results back to the user interface so the user can see the agent's progress in real-time.
5.  **Check Status:** Looks at the agent's final response in that step. Did the agent say it's finished (`<complete>`)? Does it need user input (`<ask>`)? Or does it intend to continue working?
6.  **Repeat or Stop:** If the agent hasn't finished and doesn't need input, the loop goes back to Step 1 to start the next cycle. If the agent finished or asked a question, the loop pauses or stops.

This loop is what allows the agent to perform multi-step tasks autonomously.

## How the Loop Solves Our Use Case: Listing Files

Let's revisit a simple task: User asks the agent, "List the files in my workspace."

1.  **User Message:** The user sends the message. The [API Layer (FastAPI)](01_api_layer__fastapi__.md) receives it and triggers the start of an agent run.
2.  **Loop Starts:** The **Agent Execution Loop** (specifically, the `run_agent` function) begins.
3.  **Iteration 1 - Prepare:** The loop checks billing, sees no temporary messages.
4.  **Iteration 1 - Consult Manager:** The loop calls `thread_manager.run_thread(...)`.
    *   The [ThreadManager (AgentPress)](05_threadmanager__agentpress__.md) gets the history (System Prompt + "List files").
    *   It calls the [LLM Service Interface](07_llm_service_interface_.md). The LLM sees the request and its instructions ([Agent Core & Prompt](02_agent_core___prompt_.md)).
    *   The LLM decides to use the `execute_command` tool with the command `ls -la`. It might generate a response like: "Okay, I will list the files for you. <execute-command>ls -la</execute-command>".
    *   The [ThreadManager (AgentPress)](05_threadmanager__agentpress__.md) parses this, finds the `<execute-command>` tag, and uses its `ToolRegistry` to call the `SandboxShellTool.execute_command("ls -la")` method ([Agent Tools](04_agent_tools_.md)).
    *   The tool interacts with the [Sandbox Environment](03_sandbox_environment_.md) to run `ls -la`.
    *   The tool returns the file listing (e.g., "total 0\ndrwxr-xr-x 2 user user 4096 May 10 10:00 .") to the ThreadManager.
    *   The ThreadManager saves the assistant message ("Okay, I will...") and the tool result (the listing) to the database.
    *   `run_thread` streams the results back: first the text chunk ("Okay, I will..."), then status updates about the tool starting/finishing, then the tool result chunk.
5.  **Iteration 1 - Process Results:** The loop receives these chunks from `run_thread`.
6.  **Iteration 1 - Communicate:** The loop yields these chunks back to the API layer, which sends them to the user's browser to display in real-time.
7.  **Iteration 1 - Check Status:** The loop examines the final result from this step. Let's assume the agent, after getting the tool result, decided the task was done and included `<complete>` in its final output (or maybe it used `<ask>` to ask if the user needs anything else).
8.  **Iteration 1 - Repeat or Stop:** Because the agent signalled it was done (`<complete>` or `<ask>`), the loop condition `continue_execution` becomes `False`. The loop terminates.

The agent successfully listed the files in one cycle of the loop because the task was simple. For more complex tasks, the loop would continue running, calling `run_thread` multiple times.

## Under the Hood: The `run_agent` Function

The Agent Execution Loop isn't a separate class; it's primarily implemented within the `run_agent` asynchronous function in `backend/agent/run.py`. This function orchestrates the entire process for a single agent task.

Here's a simplified view of its structure:

```python
# Simplified from backend/agent/run.py

async def run_agent(
    thread_id: str,
    project_id: str,
    stream: bool,
    # ... other parameters like model, max_iterations ...
):
    # 1. Setup: Initialize ThreadManager, add tools, get system prompt
    thread_manager = ThreadManager()
    # ... thread_manager.add_tool(...) for all needed tools ...
    system_message = { "role": "system", "content": get_system_prompt() }

    iteration_count = 0
    continue_execution = True # Loop control flag

    # --- The Main Execution Loop ---
    while continue_execution and iteration_count < max_iterations:
        iteration_count += 1
        logger.debug(f"Starting loop iteration {iteration_count}")

        # 2. Prepare: Check billing, handle temporary messages (simplified)
        # ... billing check ...
        # ... code to get temporary_message (e.g., browser state) ...

        # 3. Consult Manager: Call ThreadManager to run one step
        response_stream = await thread_manager.run_thread(
            thread_id=thread_id,
            system_prompt=system_message,
            stream=stream,
            temporary_message=temporary_message,
            # ... other LLM/processor configs ...
        )

        last_tool_call = None # Track if agent used ask/complete

        # 4. Process Results & Communicate: Handle the stream
        async for chunk in response_stream:
            # Check if the agent used a stopping tool like <ask> or <complete>
            # (Simplified logic - actual code looks for specific tags)
            if is_ask_or_complete_chunk(chunk):
                last_tool_call = get_tool_from_chunk(chunk)
                logger.info(f"Agent used stopping tool: {last_tool_call}")

            # Send the chunk back to the caller (e.g., API layer)
            yield chunk

        # 5. Check Status & Decide to Repeat/Stop
        if last_tool_call in ['ask', 'complete', 'web-browser-takeover']:
            logger.info(f"Loop stopping due to agent action: {last_tool_call}")
            continue_execution = False # Set flag to stop the loop
        else:
            logger.debug("Agent did not use stopping tool, loop continues.")
            # Check if last message was assistant (another stop condition)
            # ... (simplified check) ...
            # if await is_last_message_assistant(thread_id):
            #    continue_execution = False

    # --- Loop End ---
    logger.info(f"Agent execution loop finished after {iteration_count} iterations.")

```

**Key parts:**

*   **Initialization:** Sets up the [ThreadManager (AgentPress)](05_threadmanager__agentpress__.md) and registers the available [Agent Tools](04_agent_tools_.md).
*   **`while continue_execution ...`:** The heart of the loop. It continues as long as `continue_execution` is `True` and a maximum iteration count isn't reached.
*   **Prepare Step:** Includes necessary checks (like billing) and prepares any special input for the agent's next step (like `temporary_message`).
*   **`await thread_manager.run_thread(...)`:** This is the crucial call where the loop delegates the complex task of interacting with the LLM and executing tools to the [ThreadManager (AgentPress)](05_threadmanager__agentpress__.md).
*   **`async for chunk in response_stream:`:** The loop iterates through the results provided by the `ThreadManager`.
*   **Communicate (`yield chunk`):** Sends the progress back to the user interface.
*   **Check Status:** Examines the received chunks (especially the final ones) to see if the agent used a tool like `<ask>` or `<complete>` that signals the loop should stop.
*   **`continue_execution = False`:** Sets the flag to exit the `while` loop if a stopping condition is met.

Here's a simplified diagram showing the loop's interaction:

```mermaid
sequenceDiagram
    participant Loop as Agent Execution Loop (run_agent)
    participant TM as ThreadManager
    participant LLM_Tools as LLM & Tools (Internal to TM)
    participant UI as User Interface

    Loop->>Loop: Check Billing, Prepare Temp Msg
    Loop->>+TM: run_thread(history, temp_msg)
    TM->>+LLM_Tools: Call LLM, Parse Response, Execute Tools
    LLM_Tools-->>-TM: Generate results (text, tool output)
    TM-->>-Loop: Stream results (chunks)
    Loop->>UI: yield chunk
    Note right of Loop: User sees progress...
    Loop->>UI: yield chunk
    ...
    Loop->>Loop: Check if last chunk was <ask> or <complete>
    alt Agent Finished or Needs Input
        Loop->>Loop: Set continue_execution = False
    else Agent Continues
        Loop->>Loop: Loop repeats (continue_execution = True)
    end
```

This shows the loop (`run_agent`) repeatedly calling the `ThreadManager` (`run_thread`), processing the streamed results, sending them to the UI, and deciding whether to continue based on the agent's final action in that cycle.

## Conclusion

The **Agent Execution Loop**, primarily residing in the `run_agent` function, is the engine that drives the `suna` agent's continuous operation. It acts like a heartbeat, repeatedly coordinating the cycle of preparing input, calling the [ThreadManager (AgentPress)](05_threadmanager__agentpress__.md) to handle AI thinking and tool execution, processing the results, communicating progress, and deciding whether to continue or stop.

This loop allows the agent to tackle complex, multi-step tasks by breaking them down into manageable cycles, ensuring the agent keeps working until the job is done or requires user intervention.

We've seen that a key part of this loop is calling the LLM via the [ThreadManager (AgentPress)](05_threadmanager__agentpress__.md). How exactly does `suna` talk to different LLMs like Claude, GPT, or others? Let's explore the component responsible for this communication next.

**Next:** [Chapter 7: LLM Service Interface](07_llm_service_interface_.md)

---

Generated by [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge)