Title: Building a Coding Agent : Part 4 - A basic usable coding agent
Date: 2025-11-01
Tags: Clojure, OpenAI, LLM

We will utilize this post to get an understanding of how easy it is to build a coding agent.
You will see that there is not much required to build a coding agent, all the magic is in the LLM.
It will make you question why all these coding agent startups have such a big valuation when it is so easy for a single developer to build one.

# Write the file editing tools
We will build a few file editing tools which allow the LLM to manipulate files. This will unlock the behaviour of the coding agent.
## Read file
The first tool we will provide the LLM is a file reading tool. It can be coded and annotated for the LLM to discover as below:
```clojure
(defn
  ^{:tool {:name "read_file"
           :description "Read the contents of a given relative file path. Use this when you want to see what's inside a file. Do not use this with directory names."
           :parameters {:type "object"
                        :properties {:path {:type "string"
                                            :description "The relative path of a file in the working directory."}}
                        :required ["path"]}}}
  read-file
  [{:keys [path]}]
  (try
    (slurp path)
    (catch Exception e (str "Caught exception:" (.getMessage e)))))
```
We can immediately see the utility of this tool by asking the LLM to summarize files. An example run is below:
> **You:** What is the secret-file.txt
>
> _tool: read_file {:path secret-file.txt}_
>
> **LLM:** secret-file.txt contains a riddle:
>
> "what animal is the most disagreeable because it always says neigh?"
>
> Answer: A horse — because it always says "neigh" (pun on "nay," meaning "no").

The LLM can now read file contents and reason over the contents. Like solving the puzzle in the file `secret-file.txt`

## List directory
Next, let us give it a tool to list the files in a folder. That will free us from giving it precise file names for reasoning over.
```clojure
(defn
  ^{:tool {:name "list_files"
           :description "List files and directories at a given path. If no path is provided, lists files in the current directory."
           :parameters {:type "object"
                        :properties {:path {:type "string"
                                            :description "Optional relative path to list files from. Defaults to current directory if not provided."}}}}}
  list-files
  [{:keys [path]}]
  (let [dir (io/file (or (not-empty (str/trim path)) "."))]
    (if (.isDirectory dir)
      (str (mapv (fn [f]
                   {:name (.getName f)
                    :type (if (.isDirectory f) "directory" "file")}) (.listFiles dir)))
      "No files in the folder")))
```
Utilizing the above tool we can now summarize entire folders. Let's see a sample run:
```txt
You: What does the clojure code inside this folder do?
tool: list_files {:path }
tool: list_files {:path src}
tool: list_files {:path src/agent}
tool: read_file {:path src/agent/core.clj}
tool: read_file {:path src/agent/tools.clj}
tool: read_file {:path llm.edn}
LLM: Summary
- This is a small Clojure CLI "assistant" that talks to an LLM (via the wkok.openai-clojure client) and supports tool/function calls the LLM can request.
- The main pieces are:
  - src/agent/core.clj — orchestration, LLM calls, message loop, tool discovery/invocation.
  - src/agent/tools.clj — example tool implementations (get_current_weather, read_file, list_files).
  - llm.edn — configuration (API key, endpoint, model, impl).

What the program does (high level)
1. Reads llm.edn to get API endpoint, key, model, etc.
2. Discovers "tools" in the agent.tools namespace by looking for functions with :tool metadata (this metadata includes name, description, and parameter schema).
3. Enters a REPL-style loop:
   - Prompts the user for input.
   - Adds the user message into the conversation history.
   - Calls the LLM via openai/create-chat-completion, passing the messages and the tool list (and tool_choice "auto").
   - Inspects the LLM response. If the LLM returns tool calls (under :tool_calls), the code:
     - Maps each tool call to the actual Clojure function (from the tool registry),
     - Parses the JSON arguments,
     - Invokes the function and collects the tool output,
     - Adds each tool output back into the conversation as a tool-message (role "tool"), and
     - Repeats the LLM call with the extended message history until there are no more tool calls.
   - When a final assistant message is produced, prints it and returns to prompting the user.

Details of important functions / behavior
- get-tool-list: produces a vector of tool descriptors (for sending to the LLM) by reading :tool metadata on public functions in agent.tools.
- build-tool-registry: builds a map from tool name to the actual Clojure fn for runtime invocation.
- call-llm-api: calls openai/create-chat-completion with model, messages, tools and config keys (:api-key, :api-endpoint, :impl).
- extract-first-response: pulls the first choice's message from the API result (warns if multiple choices).
- handle-tool-call / invoke-tool: parse the JSON arguments with Cheshire and call the matching function from the registry; returns a vector of tool responses inserted into message history.
- read-user-input! / display-assistant-response! — simple CLI I/O with colored output sequences.
- parse-json-arguments uses cheshire.core/parse-string to convert the JSON argument string into Clojure data.

Example tools (agent.tools)
- get-current-weather: prints a message and returns "-25.0 C" (stubbed).
- read-file: slurps and returns the contents of a given relative file path.
- list-files: lists files/directories at a path (or current directory) and returns a vector-like string of {:name … :type …} maps.

Security / safety notes
- llm.edn contains an API key and endpoint — treat it as sensitive. The repository also contains secret-file.txt; the read_file tool can read arbitrary files, so an LLM or a user could trigger reading sensitive files.
- Tools run arbitrary code in the running process (they are Clojure functions). Be careful exposing them to untrusted LLM output/inputs.

How to run (brief)
- Use the Clojure CLI (deps.edn is present). Run the main -main in agent.core. The program will prompt you in the terminal and interact with the configured LLM.

In short: this project implements a chat loop with an LLM that can request and receive the results of local "tools" (functions), and includes a few example tools (file listing/reading and a stub weather function).

```

## Search and Replace text
Finally to allow the LLM to modify text, lets implement a search and replace tool. This tool is very primitive - only does a simple search and replace; but even with this tool the LLM can suddenly start writing code which can solve coding problems.
```clojure
(defn
  ^{:tool {:name "edit_file"
           :description "Make edits to a text file.
Replaces 'old_str' with 'new_str' in the given file. 'old_str' and 'new_str' MUST be different from each other.
If the file specified with path doesn't exist, it will be created."
           :parameters {:type "object"
                        :properties {:path {:type "string"
                                            :description "The path to the file"}
                                     :old_str {:type "string"
                                               :description "Text to search for - must match exactly and must only have one match exactly"}
                                     :new_str {:type "string"
                                               :description "Text to replace old_str with"}}
                        :required ["path" "old_str" "new_str"]}}}
  edit-file
  [{:keys [path old_str new_str]}]
  (if (= old_str new_str)
    "Error: 'old_str' and 'new_str' must be different."
    (let [file (io/file path)]
      (if (.exists file)
        (let [content (slurp file)]
          (if (not (str/includes? content old_str))
            (str "Error: '" old_str "' not found in the file.")
            (let [updated-content (str/replace content old_str new_str)]
              (spit file updated-content)
              (str "Successfully replaced '" old_str "' with '" new_str "' in " path))))
        (try
          (spit file "") ; Create an empty file if it doesn't exist
          (str "File not found. Created an empty file at " path)
          (catch Exception e (str "caught exception: " (.getMessage e))) )))))
```
Let's see a sample run with this tool available:
```txt
You: create fizzbuzz.js that I can run with Nodejs and that has fizzbuzz in it and executes it
tool: list_files {:path }
tool: edit_file {:path fizzbuzz.js, :old_str , :new_str ...}
tool: read_file {:path fizzbuzz.js}
tool: edit_file {:path fizzbuzz.js, :old_str , :new_str ...}
tool: read_file {:path fizzbuzz.js}
LLM: I created fizzbuzz.js in the project root.

How to run:
- Default (1–100): node fizzbuzz.js
- Specify n: node fizzbuzz.js 30

The file exports the fizzbuzz function if you want to require/import it in another module.

user@user-home:~/Source/code-editing-agent$ node fizzbuzz.js
1
2
Fizz
4
Buzz
Fizz
7
8
Fizz
Buzz
11
Fizz
13
14
FizzBuzz
16
17
Fizz
19
Buzz
Fizz
22
23
Fizz
Buzz
26
Fizz
28
29
FizzBuzz
31
32
Fizz
34
Buzz
Fizz
37
38
Fizz
Buzz
41
Fizz
43
44
FizzBuzz
46
47
Fizz
49
Buzz
Fizz
52
53
Fizz
Buzz
56
Fizz
58
59
FizzBuzz
61
62
Fizz
64
Buzz
Fizz
67
68
Fizz
Buzz
71
Fizz
73
74
FizzBuzz
76
77
Fizz
79
Buzz
Fizz
82
83
Fizz
Buzz
86
Fizz
88
89
FizzBuzz
91
92
Fizz
94
Buzz
Fizz
97
98
Fizz
Buzz

You: Please edit fizzbuzz.js so that it only prints until 15
tool: edit_file {:path fizzbuzz.js, :old_str ... :new_str ... }
tool: read_file {:path fizzbuzz.js}
LLM: Done — fizzbuzz.js was updated to enforce a maximum of 15. It will:

- Accept an optional n argument, but only print up to 15.
- If you request more than 15, it prints a short message to stderr like "Requested 100; printing only up to 15." and then prints results 1–15.

Run with:
- node fizzbuzz.js
- node fizzbuzz.js 20

Let me know if you want it to silently cap at 15 (no message) or to ignore command-line arguments entirely.

user@user-home:~/Source/code-editing-agent$ node fizzbuzz.js
Requested 100; printing only up to 15.
1
2
Fizz
4
Buzz
Fizz
7
8
Fizz
Buzz
11
Fizz
13
14
FizzBuzz
```
The LLM is now able to generate a full program and do changes as requested by the user. This is possible with even such a basic search replace tool - due to which the LLM rewrites entire files. Imagine the possibilities with better tools.
```txt
You: Create a congrats.js script that rot13-decodes the following string 'Pbatenghyngvbaf ba ohvyqvat n pbqr-rqvgvat ntrag!' and prints it
tool: edit_file {:path congrats.js, :old_str , :new_str ...}
tool: edit_file {:path congrats.js, :old_str , :new_str ...}
tool: read_file {:path congrats.js}
LLM: I created congrats.js. It decodes the ROT13 string and prints the result.

Run with:
- node congrats.js

If you'd like the script to accept input from the command line or a file instead, I can update it.

user@user-home:~/Source/code-editing-agent$ node congrats.js
Congratulations on building a code-editing agent!
```

As you can see writing a coding agent is fairly trivial with a powerful LLM backing it. Someone should write me a cheque for this post :D. In the next post let's look at making the agent even more autonomous.
