Title: Code Listing - Multiple MCP Servers
Date: 2025-11-11
Preview: true

**core.clj**

```clojure
(ns agent.core
  (:require
   [cheshire.core :as json]
   [clojure.edn :as edn]
   [clojure.java.io :as io]
   [clojure.pprint :as pprint]
   [clojure.string :as str]
   [wkok.openai-clojure.api :as openai]
   [agent.tools]
   [com.bunimo.clansi :as clansi]
   [agent.mcpclient :as mcpclient]
   [cheshire.core :as cheshire]))

(defn- label
  [text & attrs]
  (print (apply clansi/style text attrs) ": "))

(defn- read-user-input!
  []
  (label "You" :blue)
  (flush)
  (when-let [raw (read-line)]
    (let [message (str/trim raw)]
      (when-not (or (str/blank? message) (= message "quit"))
        {:role "user" :content message}))))

(defn-
  dbg-print
  [args]
  (label "Debug" :bright :black)
  (println args))

(defn- display-assistant-response!
  [content]
  (label "LLM" :yellow)
  (println content))

(defn- call-llm-api
  [messages config tools]
  (try
    (openai/create-chat-completion {:model (:model config)
                                    :messages messages
                                    :tools tools
                                    :tool_choice "auto"}
                                   (select-keys config [:api-key :api-endpoint :impl]))
    (catch Exception e
      (throw (ex-info "LLM API call failed" {:cause (.getMessage e)
                                             :messages messages}
                      e)))))

(defn- extract-first-response
  "The LLM call can return multiple responses. Extract the first one and print a warning if there are more than one responses"
  [response]
  (let [choices (:choices response)
        responses (mapv :message choices)]
    (when-not (= 1 (count responses))
      (pprint/pprint {:warning "Multiple responses received" :responses responses}))
    (first responses)))

(defn- add-message-to-history
  ([history message]
   (conj history message)))

(defn- parse-json-arguments
  [args]
  (json/parse-string args true))

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
      (dbg-print (str "Unknown tool" name)))))

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

(defn- read-config!
  []
  (with-open [r (io/reader "llm.edn")]
    (edn/read {:eof nil} (java.io.PushbackReader. r))))

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
  [messages config tools tools-registry]
  (let [assistant-response (-> messages
                               (call-llm-api config tools)
                               extract-first-response)
        tool-messages (handle-tool-call assistant-response tools-registry)
        new-messages (add-message-to-history messages assistant-response)]
    (if (seq tool-messages)
      (let [tool-message-history (reduce add-message-to-history new-messages tool-messages)]
        (recur tool-message-history config tools tools-registry))
      new-messages)))

(defn- get-system-prompt
  []
  {:role "developer"
   :content "You are an expert developer and can utilize tools to write programs and execute them. Please utilize the tools available to you to create and modify files as necessary."})

(defn -main
  []
  (let [config (read-config!)
        tools (get-tool-list 'agent.tools)
        servers (mcpclient/get-servers "./mcp.json")
        tool-registry (build-tool-registry 'agent.tools)
        mcp-tools (into tools (mcpclient/get-combined-tools servers))
        combined-registry (merge tool-registry (mcpclient/get-combined-tool-registry servers))]
    (doseq [tool mcp-tools]
      (dbg-print (str "Registering tool " (get-in tool [:function :name]))))
    (loop [user-message (read-user-input!)
           messages [(get-system-prompt)]]
      (when (some? user-message)
        (let [new-messages (add-message-to-history messages user-message)
              messages-including-response (get-assistant-response new-messages config mcp-tools combined-registry)
              assistant-message (:content (last messages-including-response))]
          (display-assistant-response! assistant-message)
          (recur (read-user-input!) messages-including-response))))
    (println "Exiting...")
    (doseq [server servers]
      (println "Closing " (:name server))
      (mcpclient/close-client (:client server)))))

(comment
  (add-message-to-history [] {})
  (-main))

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

**tools.clj**

```clojure
(ns agent.tools
  (:require
   [clojure.java.io :as io]
   [clojure.java.shell :as shell]
   [clojure.string :as str]))

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
