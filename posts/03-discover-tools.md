Title: Building a Coding Agent : Part 3 - Automatic discovery of tools
Date: 2025-10-31
Tags: Clojure, OpenAI, LLM

In the [previous](02-give-it-a-tool.html) post, we looked at how to provide a tool to the LLM for getting the weather.
To simplify the post we had used hard-coding and provided it directly to the LLM. In this post let's look at how we can use the Clojure metadata properties to discover tools.

# How to use Clojure metadata to discover a tool
Clojure allows tagging of all symbols with a metadata map. This allows arbitrary annotation of the code and data.
We can query the metadata attached to any symbol by:
```clojure
(pprint (meta #'+))
;; {:added "1.2",
;;  :ns #object[clojure.lang.Namespace 0x109f5dd8 "clojure.core"],
;;  :name +,
;;  :file "clojure/core.clj",
;;  :inline-arities
;;  #object[clojure.core$_GT_1_QMARK_ 0x47dd778 "clojure.core$_GT_1_QMARK_@47dd778"],
;;  :column 1,
;;  :line 986,
;;  :arglists ([] [x] [x y] [x y & more]),
;;  :doc
;;  "Returns the sum of nums. (+) returns 0. Does not auto-promote\n  longs, will throw on overflow. See also: +'",
;;  :inline
;;  #object[clojure.core$nary_inline$fn__5625 0x7de4a01f "clojure.core$nary_inline$fn__5625@7de4a01f"]}
```
Note the standard :doc, :name attributes which Clojure uses. We can add our custom metadata to all the tools we will provide the LLM. Keeping the metadata with the function makes it easier to keep the documentation and implementation in sync.
For the `get-current-weather` function we will add the following metadata:
```clojure
(defn
  ^{:tool {:name "get_current_weather"
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
  [location]
  "-25.0 C")
```
To keep things simple, we will still keep the entire metadata needed by the API as a single object which we can pass as is. We will pass in the tools to the `call-llm-api` function for it to be included in the API call.
We also need a function which will do dynamic discovery of the tools written. That can be written as below:
```clojure
(defn- get-tool-list
  "Discovers all functions in provided namespace ns with :tool metadata"
  [ns]
  (->> (ns-publics ns)
       vals
       (filter #(:tool (meta %)))
       (mapv #(hash-map :type "function" :function (:tool (meta %))))))
```
This function inspects all public symbols in the passed in namespace and filters the list to all the symbols with the `:tool` metadata attached. Using these metadata it creates the list to passed to the LLM completion call.
We also need a registry to invoke the tool when we find a matching tool call from the LLM. That can be achieved by creating a mapping of the tool name to the function symbol as below:
```clojure
(defn build-tool-registry
  "Builds a map of tool names to their corresponding functions"
  [ns]
  (->> (ns-publics ns)
       vals
       (filter #(:tool (meta %)))
       (reduce (fn [acc f]
                 (assoc acc (get-in (meta f) [:tool :name]) f))
               {})))
```
# Wiring up the tool execution
Now, that we have both of these we can change the `handle-tool-call` to utilize both of these.
```clojure
(defn handle-tool-call
  [response tools]
  (let [tool-calls (:tool_calls response)]
    (mapv (fn [tc]
            {:tool_call_id (:id tc)
             :content (invoke-tool tools tc)
             :role "tool"}) tool-calls)))
```
We will also introduce an `invoke-tool` function to better handle parameters to the call.
```clojure
(defn- invoke-tool [tools tc]
  (let [name (get-in tc [:function :name])
        fn (get tools name)]
    (println "\u001b[92mtool\u001b[0m:" name)
    (if (some? fn)
      (fn (parse-json-arguments (get-in tc [:function :arguments])))
      (str "Error calling " name))))
```
The complete code looks like:
```clojure
(ns agent.core
  (:require
   [cheshire.core :as json]
   [clojure.edn :as edn]
   [clojure.java.io :as io]
   [clojure.pprint :as pprint]
   [clojure.string :as str]
   [wkok.openai-clojure.api :as openai]
   [agent.tools]))

(defn- read-user-input!
  []
  (print "\u001b[94mYou\u001b[0m: ")
  (flush)
  (let [message (str/trim (read-line))]
    (when-not (or (str/blank? message) (= message "quit"))
      {:role "user" :content message})))

(defn- display-assistant-response!
  [content]
  (println "\u001b[93mLLM\u001b[0m:" content))

(defn- call-llm-api
  "Call the chat completion API"
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
  "Adds a message to the message history."
  ([history message]
   (conj history message)))

(defn- parse-json-arguments
  [args]
  (json/parse-string args true))

(defn- invoke-tool [tools tc]
  (let [name (get-in tc [:function :name])
        fn (get tools name)]
    (println "\u001b[92mtool\u001b[0m:" name)
    (if (some? fn)
      (fn (parse-json-arguments (get-in tc [:function :arguments])))
      (str "Error calling " name))))

(defn handle-tool-call
  [response tools]
  (let [tool-calls (:tool_calls response)]
    (mapv (fn [tc]
            {:tool_call_id (:id tc)
             :content (invoke-tool tools tc)
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
                 (assoc acc (get-in (meta f) [:tool :name]) f))
               {})))

(defn- get-assistant-response
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

(defn -main
  []
  (let [config (read-config!)
        tools (get-tool-list 'agent.tools)
        tools-registry (build-tool-registry 'agent.tools)]
    (loop [user-message (read-user-input!)
           messages []]
      (when (some? user-message)
        (let [new-messages (add-message-to-history messages user-message)
              messages-including-response (get-assistant-response new-messages config tools tools-registry)
              assistant-message (:content (last messages-including-response))]
          (display-assistant-response! assistant-message)
          (recur (read-user-input!) messages-including-response))))))
```
We have moved the tools to a separate namespace `agent.tools` to manage the code better.
# Demo

> **You:** How is the weather in San Jose?
>
> **LLM:** Which San Jose do you mean — San Jose, California (USA) or San José, Costa Rica (or another San Jose)? Also which temperature unit do you prefer: Celsius or Fahrenheit?
>
> **You:** California
>
> **LLM:** Got it — San Jose, California. Which temperature unit would you like: Celsius or Fahrenheit? If you don’t have a preference I’ll use Fahrenheit.
>
> **You:** Celsius
>
> _tool: get_current_weather_
>
> _Fetching weather for  San Jose, California, USA  in  celsius_
>
> _tool: get_current_weather_
>
> _Fetching weather for  San Jose, California, USA  in  celsius_
>
> _tool: get_current_weather_
>
> _Fetching weather for  San Jose, California, USA  in  fahrenheit_
>
> **LLM:** Sorry — the weather service returned -25.0 °C for San Jose, CA, which is almost certainly an error. I can:
>
> - Try fetching the current weather again now (Celsius or Fahrenheit), or
> - Give typical/average conditions for San Jose in late October, or
> - Help with a forecast or radar from another source.
>
> Which would you like me to do?
>
> **You:** quit

The agent works as before. Notice that the multiple calls to the tool as the LLM can't believe the value of the temperature it is getting :D.

That's it for this post. Let's look at how easy it is to build a code-editing agent in the next post.
