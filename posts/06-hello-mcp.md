Title: Building a Coding Agent : Part 6 - Hello MCP
Date: 2025-11-09
Tags: Clojure, OpenAI, LLM

Anthropic introduced a new protocol which allows standardization of how tools are written for LLMs - the [Model Context Protocol](https://modelcontextprotocol.io/). In this post lets integrate our agent with the MCP client SDK. This will allow us to integrate with publicly available MCP servers and we are freed from writing tools for every service we want to interact with.

# MCP Protocol Library
Let's integrate the official Java MCP Client library as there is no library which allows us to consume MCP servers in Clojure. The documentation for the library is at [MCP Java SDK](https://modelcontextprotocol.io/sdk/java/mcp-overview).

I have never done Java interop in Clojure before. So, this was a nice learning experience. The sequence of steps for using a MCP server is as follows:

```
Create MCP Client ->
  Initialize the Server ->
    List Tools and hand them to the LLM ->
      Call tool as requested ->
        Close the session once done
```

For getting the MCP server connection we need to create a client first. The library  supports two types of clients **Sync** and **Async**. To keep things simple let's use a sync client.

To create a sync client first we need a transport for the server. Multiple types of transports are possible but for a local server **Stdio** works fine. Let's create the transport first
```Clojure
(defn- make-transport
  [program args]
  (-> (ServerParameters/builder program)
      (.args (into-array String args))
      (.build)
      (StdioClientTransport. (McpJsonMapper/getDefault))))
```
We pass in a program and its arguments for the transport to be created. In our case we will use the default filesystem server provided by Anthropic. So, the call will be:

```Clojure
(make-transport "npx" ["-y" "@modelcontextprotocol/server-filesystem" "."])
```

Once we have a transport available, we will use it to create a sync MCP client.
```Clojure
(defn make-client
  "Create a client to the MCP server specified"
  [program args]
  (-> (make-transport program args)
      McpClient/sync
      (.build)))
```

Now we can query this server for a list of tools using:
```Clojure
(defn get-tools
  "Get the list of tools exposed by the MCP server in a format which is compatible to the OpenAI endpoint"
  [server]
  (let [result (.listTools server)
        tools (.tools result)]
    (mapv #(mcp-tool->openai-tool (tool-result->clj %)) tools)))
```
This list of tools we pass in addition to our other tools. Since, we are using the Anthropic filesystem MCP server we will unregister our tools. We will only keep the shell execution tool. We now just need to change the `invoke-tool` function to call the MCP tool if there is no coded tool available. The changed code looks like below:

```Clojure
(defn- invoke-tool [tools tc client]
  (let [name (get-in tc [:function :name])
        tool-fn (get tools name)
        args (get-in tc [:function :arguments])
        parsed-args (parse-json-arguments args)]
    (label "Tool" :green)
    (println name)
    (pprint/pprint parsed-args)
    (if (some? tool-fn)
      (tool-fn parsed-args)
      (agent.mcpclient/call-tool client name args))))
```
Our coded function expect Clojure maps but the MCP tools require JSON strings. To avoid round-tripping from JSON to Clojure and back we pass in the JSON string as is to the MCP tool call. The MCP tool function is written as:

```Clojure
(defn call-tool
  "Invoke an MCP tool with the given params"
  [client tool params]
  (let [request (McpSchema$CallToolRequest. (McpJsonMapper/getDefault) tool params)]
    (tool-result->clj (.callTool client request))))
```
The parsing of the tool result is done as below:
```Clojure
(defn- tool-result->json
  [tool-result]
  (let [mapper (McpJsonMapper/getDefault)]
    (try
      (.writeValueAsString mapper tool-result)
      (catch Exception e
        (throw (ex-info "Failed to serialize ToolResult to JSON" {:cause e}))))))

(defn- tool-result->clj
  [tool-result]
  (let [json (tool-result->json tool-result)]
    (cheshire/parse-string json true)))
```
# Demo
Now let us look at a demo run using MCP tools for the filesystem.
```txt
Debug : Registering tool run_shell_command
Debug : Registering tool read_file
Debug : Registering tool read_text_file
Debug : Registering tool read_media_file
Debug : Registering tool read_multiple_files
Debug : Registering tool write_file
Debug : Registering tool edit_file
Debug : Registering tool create_directory
Debug : Registering tool list_directory
Debug : Registering tool list_directory_with_sizes
Debug : Registering tool directory_tree
Debug : Registering tool move_file
Debug : Registering tool search_files
Debug : Registering tool get_file_info
Debug : Registering tool list_allowed_directories
You : Which libraries are used in the current project?
Tool : list_directory
{:path "."}
Tool : read_text_file
{:path "deps.edn"}
LLM : I inspected deps.edn. Libraries (with versions) used in the project are:

Main :deps
- org.clojure/clojure 1.12.0
- net.clojars.wkok/openai-clojure 0.23.0
- cheshire/cheshire 6.1.0
- com.bunimo/clansi 1.3.1
- io.modelcontextprotocol.sdk/mcp 0.15.0

Aliases (extra deps)
- lambdaisland/kaocha 1.91.1392 (under :kaocha alias)
- io.github.borkdude/quickblog (specified by git sha: 68f22a3de58af9a48f8b5aa92ee329e495d246a7) (under :quickblog alias)
- org.babashka/cli 0.3.35 (under :quickblog alias)

If you want, I can search the code to show where each library is used.
You : Can you please run the tests
Tool : run_shell_command
{:command "clojure -M:kaocha"}
LLM : I ran the test suite. Output summary:

3 tests, 3 assertions, 0 failures.

Would you like the full test output or a breakdown of which tests ran?
You : Show the full test output
Tool : run_shell_command
{:command
 "clojure -M:kaocha --reporter documentation 2>&1 | sed -n '1,400p'"}
LLM : Here is the full test output (including the test names and the final summary):

--- unit (clojure.test) ---------------------------
agent.mcpclient-test
  mcp-tool->openai-tool-missing-name-test
  mcp-tool->openai-tool-test
  mcp-tool->openai-tool-defaults-test

3 tests, 3 assertions, 0 failures.
You : quit
Exiting...
```
The LLM is able to generate a summary of the libraries used in the project using the Filesystem MCP server. It also uses our `run_shell_command` tool to run the tests in the project.

The full listing of the code is [here](06-full-listing.html). In addition to Clojure you will also need Node installed on your machine for the ability to run MCP servers.
