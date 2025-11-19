Title: Code Listing - Handling Commands
Date: 2025-11-19
Preview: true

**core.clj**

```clojure
(ns agent.core
  (:require
   [agent.mcpclient :as mcpclient]
   [agent.state :as state]
   [agent.tools]
   [agent.utils :as utils :refer [dbg-print display-assistant-response! label
                                  parse-json-arguments read-user-input!]]
   [clojure.pprint :as pprint]
   [wkok.openai-clojure.api :as openai]))

(defn- call-llm-api
  [messages config tools]
  (try
    (openai/create-chat-completion {:model (:model config)
                                    :messages messages
                                    :tools tools
                                    :tool_choice "auto"}
                                   (select-keys config [:api-key :api-endpoint :impl]))
    (catch Exception e
      (dbg-print (get-in (parse-json-arguments (:body (ex-data e))) [:error :message]))
      (throw (ex-info "LLM API call failed" {:cause e
                                             :messages messages}
                      e)))))

(defn- extract-first-response
  "The LLM call can return multiple responses. Extract the first one and print a warning if there are more than one responses"
  [{:keys [choices]}]
  (let [responses (mapv :message choices)]
    (when-not (= 1 (count responses))
      (pprint/pprint {:warning "Multiple responses received" :responses responses}))
    (first responses)))

(defn- invoke-tool [tools tc]
  (let [name (get-in tc [:function :name])
        tool-fn (get tools name)
        args (get-in tc [:function :arguments])
        parsed-args (parse-json-arguments args)]
    (label "Tool" :green)
    (println name)
    (pprint/pprint parsed-args)
    (if (some? tool-fn)
      (tool-fn args)
      (dbg-print "Unknown tool" name))))

(defn- encode-tool-response
  [response]
  (str response))

(defn handle-tool-call
  [response tools]
  (let [tool-calls (:tool_calls response)]
    (mapv (fn [tc]
            {:tool_call_id (:id tc)
             :content (encode-tool-response (invoke-tool tools tc))
             :role "tool"}) tool-calls)))

(defn- get-tool-list
  "Discovers all functions in provided namespace ns with :tool metadata"
  [ns]
  (->> (ns-publics ns)
       vals
       (filter #(:tool (meta %)))
       (mapv #(hash-map :type "function" :function (:tool (meta %))))))

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

(defn- get-assistant-response
  "Recursively gets assistant responses handling tool calls as needed"
  [{:keys [history config tools tools-registry] :as state}]
  (let [start-time (System/currentTimeMillis)
        api-response (call-llm-api history config tools)
        assistant-response (extract-first-response api-response)
        tool-messages (handle-tool-call assistant-response tools-registry)
        new-messages (state/add-message-to-history history assistant-response)]
    (if (seq tool-messages)
      (let [tool-message-history (reduce state/add-message-to-history new-messages tool-messages)]
        (recur (assoc state :history tool-message-history)))
      {:history new-messages
       :usage (select-keys (:usage api-response) [:completion_tokens :prompt_tokens :total_tokens])
       :duration (/ (- (System/currentTimeMillis) start-time) 1000.0)})))

(defn -main
  []
  (let [system-prompt (state/get-system-prompt)
        prompts {:system-prompt system-prompt}
        config (utils/read-config! "llm-azure.edn")
        tools (get-tool-list 'agent.tools)
        servers (mcpclient/get-servers "./mcp.json")
        tool-registry (build-tool-registry 'agent.tools)
        mcp-tools (into tools (mcpclient/get-combined-tools servers))
        combined-registry (merge tool-registry (mcpclient/get-combined-tool-registry servers))
        initial-state {:history [system-prompt]
                       :prompts prompts
                       :config config
                       :tools mcp-tools
                       :tools-registry combined-registry
                       :next-state :user}]
    (doseq [tool mcp-tools]
      (dbg-print "Registering tool " (get-in tool [:function :name])))
    (loop [{:keys [next-state] :as current-state} (state/handle-user-input! initial-state (read-user-input!))]
      (cond
        (= next-state :quit)
        (do
          (println "Exiting")
          (doseq [server servers]
            (println "Closing " (:name server))
            (mcpclient/close-client (:client server))))

        (= next-state :llm)
        (let [{:keys [history] :as response} (get-assistant-response current-state)]
          (display-assistant-response! response)
          (recur (state/handle-user-input! (assoc current-state :history history) (read-user-input!))))

        (= next-state :user)
        (recur (state/handle-user-input! current-state (read-user-input!)))))))

(comment
  (-main)
  )

```

**mcpclient.clj**

```clojure
(ns agent.mcpclient
  (:require [cheshire.core :as cheshire]
            [clojure.pprint :as pprint])
  (:import (io.modelcontextprotocol.client.transport ServerParameters StdioClientTransport)
           (io.modelcontextprotocol.json McpJsonMapper)
           (io.modelcontextprotocol.spec McpSchema$CallToolRequest)
           (io.modelcontextprotocol.client McpClient)))

(defn- make-transport
  [program args]
  (-> (ServerParameters/builder program)
      (.args (into-array String args))
      (.build)
      (StdioClientTransport. (McpJsonMapper/getDefault))))

(defn make-client
  "Create a client to the MCP server specified"
  [program args]
  (-> (make-transport program args)
      McpClient/sync
      (.build)))

(defn close-client
  "Stop the MCP server gracefully"
  [client]
  (.closeGracefully client))

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

(defn- mcp-tool->openai-tool
  "Convert the MCP tool metadata to a OpenAI compatible metadata"
  [{:keys [name description inputSchema] :as mcp-tool}]
  (when-not name
    (throw (ex-info "tool missing :name" {:tool mcp-tool})))
  {:type "function"
   :function {:name (str name)
              :description (or description "")
              :parameters (or inputSchema {})}})

(defn- get-tools
  "Get the list of tools exposed by the MCP server in a format which is compatible to the OpenAI endpoint"
  [client]
  (let [result (.listTools client)
        tools (.tools result)]
    (mapv #(mcp-tool->openai-tool (tool-result->clj %)) tools)))

(defn call-tool
  "Invoke an MCP tool with the given params"
  [client tool params]
  (let [request (McpSchema$CallToolRequest. (McpJsonMapper/getDefault) tool params)]
    (tool-result->clj (.callTool client request))))

(defn- read-mcp-json
  [path]
  (let [content (slurp path)
        config (cheshire/parse-string content true)]
    (or (:mcpServers config) [])))

(defn get-servers
  "Get a vector of clients connected to the MCP servers specified in the config file passed in"
  [path]
  (let [servers (read-mcp-json path)]
    (mapv (fn [[server-name server-config]]
            (let [command (:command server-config)
                  args (or (:args server-config) [])]
              (println "Connecting " server-name)
              {:name server-name
               :client (make-client command args)}))
          servers)))

(defn- build-tool-registry
  [client]
  (let [result (.listTools client)
        tools (.tools result)
        tool-metadata (mapv #(mcp-tool->openai-tool (tool-result->clj %)) tools)]
    (reduce (fn [acc f]
              (let [name (get-in f [:function :name])]
                (assoc acc name (fn [args] (call-tool client name args)))))
            {} tool-metadata)))

(defn get-combined-tools
  "Given a list of servers, generate a combined list of tools from all servers"
  [servers]
  (mapcat (fn [{:keys [client]}]
            (get-tools client))
          servers))

(defn get-combined-tool-registry
  "Given a list of servers, generate a combined tool registry from all servers"
  [servers]
  (apply merge {} (map (comp build-tool-registry :client) servers)))

(comment
  (let [client (make-client "npx" ["-y" "@modelcontextprotocol/server-filesystem" "."])]
    (clojure.pprint/pprint (build-tool-registry client))
    (close-client client))
  (read-mcp-json "./mcp.json"))

```

**state.clj**

```clojure
(ns agent.state
  (:require
   [agent.utils :as utils]
   [clojure.pprint :as pprint]
   [clojure.string :as str]))


(defn add-message-to-history
  ([history message]
   (conj history message)))

(defn get-system-prompt
  []
  {:role "developer"
   :content "As an expert developer, utilize the full suite of available tools to write, execute, and debug programs effectively. When required, create new files or modify existing ones as necessary to accomplish your development tasks. Be thorough in your approach, ensuring each step and modification is well-documented and logically sound. While generating responses to the user prefer using Markdown for the output."})

(defn handle-user-input!
  [{:keys [history prompts] :as state} input]
  (if
   (str/starts-with? input "/")
    (let [args (str/split input #" ")
          command (-> (first args) (subs 1) str/lower-case keyword)]
      (cond (= command :quit)
            (assoc state :next-state :quit)

            (= command :clear)
            (do
              (println "Clearing history")
              (assoc state :next-state :user
                     :history [(:system-prompt prompts)]))

            (= command :debug)
            (do
              (println "====== Current State ======")
              (pprint/pprint state)
              (println "===========================")
              (assoc state :next-state :user))

            (= command :model)
            (let [model-name (second args)
                  config (utils/read-config! (str "llm-" model-name ".edn"))]
              (if (some? config)
                (do
                  (println "Switching model to: " model-name)
                  (assoc state :config config :next-state :user))
                state))

            :else
            (assoc state :next-state :user)))
    (assoc state :next-state :llm
           :history (add-message-to-history history {:role "user" :content input}))))

```

**tools.clj**

```clojure
(ns agent.tools
  (:require
   [clojure.java.io :as io]
   [clojure.java.shell :as shell]
   [clojure.string :as str]))

#_{:clojure-lsp/ignore [:clojure-lsp/unused-public-var]}
(defn
  ^{:tool-unused {:name "get_current_weather"
           :description "Retrieves the current weather information for a specified location"
           :parameters {:type "object"
                        :properties {:location {:type "string"
                                                :description "The city and state/country for which to get weather information"}
                                     :unit {:type "string"
                                            :enum ["celsius" "fahrenheit" "kelvin"]
                                            :description "Temperature unit for the response"
                                            :default "celsius"}}
                        :required ["location"]}}}
  get-current-weather
  [_]
  "-25.0 C")

#_{:clojure-lsp/ignore [:clojure-lsp/unused-public-var]}
(defn
  ^{:tool-unused {:name "read_file"
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

#_{:clojure-lsp/ignore [:clojure-lsp/unused-public-var]}
(defn
  ^{:tool-unused {:name "list_files"
           :description "List files and directories at a given path. If no path is provided, lists files in the current directory."
           :parameters {:type "object"
                        :properties {:path {:type "string"
                                            :description "Optional relative path to list files from. Defaults to current directory if not provided."}}}}}
  list-files
  [{:keys [path] :or {path "."}}]
  (let [dir (io/file (or (not-empty (str/trim path)) "."))]
    (if (.isDirectory dir)
      (mapv (fn [f]
              {:name (.getName f)
               :type (if (.isDirectory f) "directory" "file")}) (.listFiles dir))
      "No files in the folder")))

#_{:clojure-lsp/ignore [:clojure-lsp/unused-public-var]}
(defn
  ^{:tool-unused {:name "edit_file"
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

#_{:clojure-lsp/ignore [:clojure-lsp/unused-public-var]}
(defn
  ^{:tool {:name "run_shell_command"
           :description "Run the provided bash shell command and get the output"
           :parameters {:type "object"
                        :properties {:command {:type "string"
                                               :description "Command to be executed"}}
                        :required ["command"]}}}
  run-shell-command
  [{:keys [command]}]
  (try
    (let [{:keys [out err exit]} (shell/sh "bash" "-c" command)]
      (if (zero? exit)
        out
        (str "Error: " err)))
    (catch Exception e
      (str "Exception occurred: " (.getMessage e)))))

```

**utils.clj**

```clojure
(ns agent.utils
  (:require
   [cheshire.core :as json]
   [clojure.edn :as edn]
   [clojure.java.io :as io]
   [clojure.string :as str]
   [com.bunimo.clansi :as clansi]
   [markdown-term.core :refer [render-markdown->ansi]]))

(defn read-config!
  [config]
  (try
    (with-open [r (io/reader config)]
      (edn/read {:eof nil} (java.io.PushbackReader. r)))
    (catch java.io.FileNotFoundException _
      (println "Unable to read:" config)
            nil)))

(defn label
  [text & attrs]
  (print (apply clansi/style text attrs) ": "))

(defn read-user-input!
  []
  (label "You" :blue)
  (flush)
  (if-let [raw (read-line)]
    (let [message (str/trim raw)]
      (if
       (str/blank? message)
        (recur)
        message))
    "/quit"))

(defn
  dbg-print
  [& args]
  (label "Debug" :bright :black)
  (apply println args))

(defn parse-json-arguments
  [args]
  (json/parse-string args true))

(defn display-assistant-response!
  [{:keys [history usage duration] :as response}]
  (let [llm-message (last history)
        content (:content llm-message)
        reasoning (:reasoning llm-message)]
    (when (some? reasoning)
      (label "Reasoning" :gray)
      (println reasoning))
    (label "LLM" :yellow)
    (println (render-markdown->ansi content)))
  (dbg-print usage "Response took - " duration " s"))

```
