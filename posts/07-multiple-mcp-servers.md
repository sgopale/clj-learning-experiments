Title:  Building a Coding Agent : Part 7 - Multiple MCP Servers
Date: 2025-11-11
Tags: Clojure, OpenAI, LLM

In the previous post we looked at connecting to a MCP server - specifically to the Anthropic file system MCP server. Let's extend the functionality of the coding agent to talk to multiple configurable MCP servers.
# mcp.json configuration
We will reuse the same format that Claude Code uses for configuring MCP servers. The file is of the format
```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "/Users/username/Desktop",
        "/path/to/other/allowed/dir"
      ]
    },
    "sequential-thinking": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-sequential-thinking"
      ]
    }
  }
}
```
Each MCP server is specified with a path to an executable. We will only deal with local MCP servers, leaving remote MCP servers for another post.

# Dealing with multiple MCP servers
We had directly invoked the tool call on the MCP client connection as we had a single MCP server and did not need to disambiguate between calls. While supporting multiple MCP servers we need to route the tool call to the correct MCP server. For doing this lets remove the direct MCP tool call and utilize the tool registry to direct the calls. Recall that we had built a tool registry to register function based tool calls. The registry was a map of the form `name -> function`. Let us keep the same structure, except we will bind the function to a dynamic function which calls the MCP tool as appropriate.

We will bind each MCP tool to a function of the form. The server and name are already available when creating the binding. The args are passed in by the `invoke-tool` call.
```clojure
(fn [args] (call-tool server name args))

(defn- build-tool-registry
  [client]
  (let [result (.listTools client)
        tools (.tools result)
        tool-metadata (mapv #(mcp-tool->openai-tool (tool-result->clj %)) tools)]
    (reduce (fn [acc f]
              (let [name (get-in f [:function :name])]
                (assoc acc name (fn [args] (call-tool client name args)))))
            {} tool-metadata)))
```
Since, our regular tool requires parsed json arguments - we will change that function binding to convert the json string args into a Clojure map.
```clojure
(defn build-tool-registry
  "Builds a map of tool names to their corresponding functions"
  [ns]
  (->> (ns-publics ns)
       vals
       (filter #(:tool (meta %)))
       (reduce (fn [acc f]
                 (let [name (get-in (meta f) [:tool :name])]
                   (assoc acc name (fn [args] (f (parse-json-arguments args))))))
               {})))
```
# Demo
We will force the LLM to use the thinking tools with the following prompt - "Help me write a factorial function in CLJS. Please think a bit before writing the code. Please use the tools available to you."
With this prompt the LLM uses both the MCP servers available to it.
```txt
----- Registering MCP Servers ----
Connecting  :sequential-thinking
Connecting  :filesystem
Debug : Registering tool run_shell_command
Debug : Registering tool sequentialthinking
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
You : Help me write a factorial function in CLJS. Please think a bit before writing the code. Please use the tools available to you.
----- Use of the Sequential Thinking MCP server ----
Tool : sequentialthinking
{:thoughtNumber 1,
 :totalThoughts 6,
 :thought
 "I need to plan before coding. Decide which factorial variants to provide and consider CLJS-specific constraints: recursion vs tail recursion, JS number limits, validation of input. Provide clear examples and safe checks.",
 :nextThoughtNeeded true}
Tool : sequentialthinking
{:thoughtNumber 2,
 :totalThoughts 6,
 :thought
 "I should implement two versions: a simple recursive factorial and a tail-recursive one using ClojureScript's recur or loop/recur. CLJS supports recur but needs to be in tail position. Also consider using BigInt for large factorials? JavaScript BigInt exists but CLJS interop may be needed – use js/BigInt. Provide an arbitrary-precision variant using BigInt and one using numbers. Also include input validation: ensure n is integer >= 0.",
 :nextThoughtNeeded true}
Tool : sequentialthinking
{:thoughtNumber 3,
 :totalThoughts 6,
 :thought
 "Decide file structure: write a CLJS file with namespace, docstrings, and exported functions. Possibly provide examples/tests. Use cljs.node or browser? Keep generic. I'll create src/factorial/core.cljs with functions: factorial (number, tail-recursive using loop/recur), factorial-bigint using js/BigInt and recursive loop. Add validation functions and small test printouts in -main for Node. Use ^:export to expose functions.",
 :nextThoughtNeeded true}
----- Use of the filesystem server ----
Tool : create_directory
{:path "src/factorial"}
Tool : write_file
{:path "src/factorial/core.cljs",
 :content ...}
Tool : edit_file
{:path "src/factorial/core.cljs",
 :edits ...}

LLM : Thanks — I thought through options before writing code and used the workspace to create a small CLJS module with two implementations and examples.

What I considered
- Provide a safe, tail-recursive implementation for ordinary JS Numbers (fast, but limited by IEEE-754 and 2^53 precision).
- Provide a BigInt-based implementation for arbitrarily large factorials using JS BigInt via interop (requires modern JS engine or a polyfill).
- Validate input (non-negative integer).
- Use loop/recur so recursion is tail-recursive and safe for reasonably large n.

What I wrote
I created src/factorial/core.cljs with:
- factorial: tail-recursive using loop/recur; returns a JS Number (double).
- factorial-bigint: tail-recursive using js/BigInt; returns a BigInt.
- run-examples and -main for quick testing.

File: src/factorial/core.cljs
(You can open this file in your project — I created it for you.)

Key functions (contents in the file):

ns factorial.core

1) factorial
- Signature: (factorial n)
- Behavior: checks that n is an integer and non-negative, then computes n! with loop/recur, returning a JS Number.
- Notes: Overflow / loss of integer precision beyond about 2^53-1 / ~170! for magnitude.

2) factorial-bigint
- Signature: (factorial-bigint n)
- Behavior: converts n to a JS BigInt (js/BigInt), checks non-negative, computes n! with loop/recur and BigInt arithmetic, returns a BigInt.
- Notes: Accepts numbers or string representations convertible to BigInt; requires environment support for BigInt (modern Node, browsers).

3) run-examples / -main
- Small helper that prints sample results:
  - factorial 5 => 120
  - factorial-bigint 20 => 2432902008176640000n (printed via str)

Usage examples (REPL)
- (require 'factorial.core)
- (factorial 5)           => 120
- (factorial-bigint 20)   => 2432902008176640000n
- (str (factorial-bigint 100)) => (string of 100!)

Notes, caveats, and tips
- Use factorial for small n where performance/primitive numbers are fine.
- Use factorial-bigint for large n and when exact integer results are required.
- BigInt interop: js/BigInt is used. On older runtimes without BigInt, you'll get an error; a BigInt polyfill or alternate big-integer library would be required.
- If you want a pure CLJS arbitrary-precision library (works everywhere), I can rewrite factorial-bigint to use a JS big-integer library (like big-integer.js via npm) or a ClojureScript big-integer library.

If you’d like, I can:
- Add tests (cljs.test).
- Provide project config / build sample (shadow-cljs or cljs.main) to run on Node or in browser.
- Add a memoized or lazy-stream implementation for many repeated factorial calls.

Would you like me to open the file contents here, add tests, or show how to compile/run it with shadow-cljs / cljs.main?
```

The full listing of the code is [here](07-full-listing.html).
