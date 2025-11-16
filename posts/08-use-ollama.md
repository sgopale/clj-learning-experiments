Title:  Building a Coding Agent : Part 8 - Using Local models
Date: 2025-11-16
Tags: Clojure, Ollama, LLM

# Using Ollama models
Now that we built a decently working coding agent, let us see if we can make it run against a local model. There are a lot of claims on the internet about local models being the future of this space. Let us validate if that claim holds true.

We will use [Ollama](https://ollama.ai) to run a local model. Luckily for us it supports the OpenAI API out of the box. So, if you connect to http://localhost:11434/v1, you can reuse the agent that we already wrote to try out the different models.

My setup is a AMD Ryzen 1600 desktop with 32GB RAM and a RTX 3060 12 GB. So, I am both CPU and GPU poor :D.

# Llama 3.2 (3B)

The first one I tried is Llama 3.2 (3b). We have to look for models which support tool calling for the agent to work. For example, the Gemma models from Google do not support tool calling and so cannot be used with our agent. The model download was relatively small for Llama 3.2 - around 2 GB. It was decently fast also due to it fitting in the VRAM of my device completely. I could get responses within 2 seconds. However, that was where the positives ended. The model hallucinates tool calls and calls them incorrectly - mostly it is unable to generate the correct tool call format. I gave it a task of writing a Javascript FizzBuzz program and this is what I got.
```text
You : Write me a FizzBuzz Javascript program with the tools available to you
Tool : run_shell_command
{:command
 "node -e <garbage generated>}
LLM : It seems like the shell command is not executed correctly.

Here's a simplified version of FizzBuzz written in JavaScript using a conventional syntax:

function fizzBuzz(n) {
    let output = '';
    for (let i = 1; i <= n; i++) {
        if (i % 3 === 0 && i % 5 === 0) {
            output += 'FizzBuzz ';
        } else if (i % 3 === 0) {
            output += 'Fizz ';
        } else if (i % 5 === 0) {
            output += 'Buzz ';
        } else {
            output += i + ' ';
        }
    }
    console.log(output);
}

fizzBuzz(20);

This function prints out the numbers from 1 to n with a twist: multiples of three are replaced by "Fizz", multiples of five by "Buzz", and multiples of both by "FizzBuzz".

To call this function, you can run it in your environment:

javascript
function fizzBuzz(n) {
    let output = '';
    for (let i = 1; i <= n; i++) {
        if (i % 3 === 0 && i % 5 === 0) {
            output += 'FizzBuzz ';
        } else if (i % 3 === 0) {
            output += 'Fizz ';
        } else if (i % 5 === 0) {
            output += 'Buzz ';
        } else {
            output += i + ' ';
        }
    }
    console.log(output);
}

fizzBuzz(20);
Debug : {:completion_tokens 334, :prompt_tokens 227, :total_tokens 561} Response took -  3.772  s
You : quit
```
So, this model is not performing well. It could not even call the tool to read/write files correctly.

# Qwen 2.5 coder (1.5B)
I had heard a lot of great things about the Qwen coding models. So, I tried the smallest of those next - the one with 1.5B parameters. Ollama downloads around 1GB of model data for this to run. I asked it what the program in the current folder does?
```text
You : What does the program in the current folder do?
LLM : The program in the current folder is a Python script named <script-name>, and it uses the run_shell_command function from the tools section to execute the command find . -type f -exec cat {} + > results.txt.

json
{
  "name": "run_shell_command",
  "arguments": {
    "command": "find ./ -type f -exec cat {} + > results.txt"
  }
}
Debug : {:completion_tokens 91, :prompt_tokens 2405, :total_tokens 2496} Response took -  3.842  s
You : quit
```
No dice. It is also not able to run any tools. Maybe I should try with a larger model.

# Qwen 3 VL (8b)
Maybe the size of the model is an issue. Let us try with a larger model. The latest Qwen 3 VL (8b) parameters.
This one manages tool calls perfectly and is able to answer what the code in the current project does.
```text
You : What does the program in the current folder do?
Tool : list_directory
{:path ". "}
Tool : list_allowed_directories
{}
Tool : list_directory
{:path "/home/sgopale/Source/code-editing-agent"}
Tool : list_directory
{:path "/home/sgopale/Source/code-editing-agent/src"}
Tool : list_directory
{:path "/home/sgopale/Source/code-editing-agent/src/agent"}
Tool : read_text_file
{:path "/home/sgopale/Source/code-editing-agent/src/agent/core.clj"}
Reasoning : Okay, so the user asked what the program in the current folder does. After listing the directories and checking the contents, I found that the main program is likely in the 'agent' directory's core.clj file. The tool response shows that core.clj is a Clojure script.

Looking at the content, it defines the namespace `agent.core`, which includes functions for interacting with LLM APIs, handling tool calls, and a main entry point `-main`. The code uses libraries like `agent.mcpclient`, `agent.state`, and `agent.tools`, which suggest it's an assistant or agent system that communicates with an LLM (like OpenAI) and handles tool invocations.

Specifically, the `-main` function initializes the system by reading configuration from `llm-ollama.edn`, getting tools from `agent.tools`, and processing user input. It uses a loop to handle user interactions, calling the LLM for responses, and managing tool calls. The code includes features like displaying responses, handling tool registration, and exiting the application.

The file also has a commented `-main` call, indicating it's meant to be run as the main entry point. The program seems to be an assistant that processes user inputs through an LLM, executes tools (like API calls or functions), and manages the conversation flow.

So putting it all together, the program is a CLI-based assistant that uses LLMs and tools to interact with users, likely part of a larger application for AI-assisted tasks.
LLM : The program in the current folder is a Clojure-based interactive agent/assistant system designed to interact with LLMs (Large Language Models) and execute tools based on user input. Below is a breakdown of its purpose and functionality:

Key Features & Purpose


1. LLM Integration


- Connects to external LLM APIs (e.g., Ollama, OpenAI) via wkok.openai-clojure.api and agent.mcpclient.
- Handles tool invocations (e.g., functions like mcpclient/get-servers) and processes responses.
2. Tool-Based Workflow


- Registers and executes custom tools (e.g., from agent.tools namespace).
- Dynamically manages tool responses (e.g., parsing tool calls, handling errors).
- Supports nested interactions where LLM output triggers tool execution, which then feeds back into the conversation.
3. User Interaction Loop


- Starts with a system prompt and reads user input via read-user-input!.
- Processes inputs through a loop:


- Calls the LLM to generate responses.
- Handles tool calls (e.g., invoke-tool).
- Displays results to the user (display-assistant-response!).
- Continues until the user exits (:quit).
4. Configuration & State Management


- Loads configuration from llm-ollama.edn (e.g., API keys, endpoints).
- Tracks conversation history (:history) and tool registry (:tools-registry).
- Maintains state across interactions (e.g., state/handle-user-input!).
5. Debugging & Logging


- Uses dbg-print for debug logs (e.g., tool registration, errors).
- Prints warnings for edge cases (e.g., multiple LLM responses).

Intended Use Case

This program is part of a CLI-friendly AI assistant system, likely used for:


- Building chatbots or interactive workflows.
- Testing LLM tool integration (e.g., connecting to APIs like Ollama).
- Processing user queries with dynamic tool chains (e.g., "Explain X, then fetch data from API Y").
It’s designed to be configurable (via llm-ollama.edn), extendable (by adding tools to agent.tools), and interactive (via read-user-input!).

Summary

The program is an LLM-driven assistant that:


- Interacts with users through a loop.
- Executes tools dynamically (e.g., API calls, custom functions).
- Handles errors and logs for debugging.
- Is configured via llm-ollama.edn and tools in agent.tools.
It’s ideal for AI-powered workflows requiring dynamic tool execution and natural language interactions. To run it, call (-main) in the Clojure REPL.
Debug : {:completion_tokens 1140, :prompt_tokens 7151, :total_tokens 8291} Response took -  44.885  s
```

Finally success. This model works reliably with tool calling and does a decent job of summarizing what the code does. However, it is too slow for me. It took approximately 45 seconds to process this prompt. Comparing it with an Azure hosted GPT-5 mini model, the online model takes 11 seconds. The online model is running at the cheapest config possible so can be faster if we choose to pay more.

So, even if the future of AI agents is local models. I believe we are not in that future yet, at least for coding agents which can call tools. Maybe these models are good for answering questions about code. I have not tried them for that use case. I will stick to online hosted models for my workflows.
