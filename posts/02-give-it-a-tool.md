Title: Building a Coding Agent : Part 2 - Adding a tool to the agent
Date: 2025-10-30
Tags: Clojure, OpenAI, LLM

In the [previous](01-build-the-loop.html) post we looked at a simple LLM loop which the user could chat with. However, the LLM did not have any way to fetch external information. Like when we asked it for the weather in Paris it would give a general sense of the weather based on the date - which it was aware of. However, it could not give any precise information. In this post, let's fix that by providing it with a `get_current_weather` tool.

# OpenAI tool documentation
In the previous post we have already seen how the LLM responds and how the user messages are tagged in the history.
For an LLM to be aware of tools those need to be passed into the completions API along with some metadata describing their use.

For the get_current_weather tool, this is the metadata format prescribed by OpenAI.
```json
{
    "type": "function",
    "name": "get_current_weather",
    "description": "Retrieves current weather for the given location.",
    "parameters": {
        "type": "object",
        "properties": {
            "location": {
                "type": "string",
                "description": "City and country e.g. Bogotá, Colombia"
            },
            "units": {
                "type": "string",
                "enum": ["celsius", "fahrenheit"],
                "description": "Units the temperature will be returned in."
            }
        },
        "required": ["location", "unit"],
        "additionalProperties": false
    },
    "strict": true
}
```
We need to give it a `description`, a `name`, and the parameters it expects in the form of a JSON object.

# How tools operate?
Now that the LLM has a tool to get the weather available, let's look at how it is invoked.
Whenever the LLM needs the tool to be executed, it generates a special message of the form
```json
{
    "id": "fc_67890abc",
    "call_id": "call_67890abc",
    "type": "function_call",
    "name": "get_current_weather",
    "arguments": "{\"location\":\"Bogotá, Colombia\"}"
}
```
There can be multiple such tool calls. Each call is identified by a unique `call_id`. We can process each tool call and return the output in a special message like below.
The call_id is to match outputs of the tool calls to the appropriate LLM tool call request.
```json
{
  "type": "function_call_output",
  "call_id": "call_67890abc",
  "content": "-26 C",
  "role": "tool"
}
```
Note the `tool` role instead of the `user` or `assistant` role.

# Handle the weather query
Now armed with the above information, we can change our LLM call to the following.
```clojure
(openai/create-chat-completion {:model (:model config)
                                    :messages messages
                                    :tools
                                     [{:type     "function"
                                       :function {:name        "get_current_weather"
                                                  :description "Get the current weather in a given location"
                                                  :parameters
                                                  {:type       "object"
                                                   :properties {:location {:type        "string"
                                                                           :description "The city and state, e.g. San Francisco, CA"}
                                                                :unit     {:type "string"
                                                                           :enum ["celsius" "fahrenheit"]}}}}}]
                                     :tool_choice "auto"}
                                   (select-keys config [:api-key :api-endpoint :impl]))
```
For now, we will hard code the metadata and the handling of the tool call.
```clojure
(defn- handle-tool-call
  [response]
  (let [tool-calls (:tool_calls response)]
    (mapv (fn [tc]
            {:type "function_call_output"
             :tool_call_id (:id tc)
             :content "-26 C"
             :role "tool"}) tool-calls))
  )
```

And we change the main loop to handle the tool calls also.
```clojure
(let [assistant-response (-> messages
                               (call-llm-api config tools)
                               extract-first-response)
        tool-messages (handle-tool-call assistant-response)
        new-messages (add-message-to-history messages assistant-response)]
    (if (seq tool-messages)
      (let [tool-message-history (reduce add-message-to-history new-messages tool-messages)]
        (recur tool-message-history config tools))
      new-messages)))
```

The final code looks like:
```clojure
(ns agent.core
  (:require
   [clojure.edn :as edn]
   [clojure.java.io :as io]
   [clojure.pprint :as pprint]
   [clojure.string :as str]
   [wkok.openai-clojure.api :as openai]))

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
  [messages config]
  (try
    (openai/create-chat-completion {:model (:model config)
                                    :messages messages
                                    :tools
                                    [{:type     "function"
                                      :function {:name        "get_current_weather"
                                                 :description "Get the current weather in a given location"
                                                 :parameters
                                                 {:type       "object"
                                                  :properties {:location {:type        "string"
                                                                          :description "The city and state, e.g. San Francisco, CA"}
                                                               :unit     {:type "string"
                                                                          :enum ["celsius" "fahrenheit"]}}}}}]
                                    :tool_choice "auto"}
                                   (select-keys config [:api-key :api-endpoint :impl]))
    (catch Exception e
      (throw (ex-info "LLM API call failed" {:cause (.getMessage e)
                                             :messages messages}
                      e)))))

(defn- extract-first-response
  "The LLM call can return multiple responses. Extract the first one and throw an exception if there are more than one responses"
  [response]
  (let [choices (:choices response)
        responses (mapv :message choices)]
    (pprint/pprint responses)
    (when-not (= 1 (count responses))
      (throw (ex-info "Expected exactly one response" {:responses responses})))
    (first responses)))

(defn- add-message-to-history
  "Adds a message to the message history."
  ([history message]
   (conj (or history []) message)))

(defn- handle-tool-call
  [response]
  (let [tool-calls (:tool_calls response)]
    (mapv (fn [tc]
            {:type "function_call_output"
             :tool_call_id (:id tc)
             :content "-26 C"
             :role "tool"}) tool-calls)))

(defn- read-config!
  []
  (with-open [r (io/reader "llm.edn")]
    (edn/read {:eof nil} (java.io.PushbackReader. r))))

(defn- get-assistant-response
  "Recursively gets assistant responses handling tool calls as needed"
  [messages config]
  (let [assistant-response (-> messages
                               (call-llm-api config)
                               extract-first-response)
        tool-messages (handle-tool-call assistant-response)
        new-messages (add-message-to-history messages assistant-response)]
    (if (seq tool-messages)
      (let [tool-message-history (reduce add-message-to-history new-messages tool-messages)]
        (recur tool-message-history config))
      new-messages)))

(defn -main
  []
  (let [config (read-config!)]
    (loop [user-message (read-user-input!)
           messages []]
      (when (some? user-message)
        (let [new-messages (add-message-to-history messages user-message)
              messages-including-response (get-assistant-response new-messages config)
              assistant-message (:content (last messages-including-response))]
          (display-assistant-response! assistant-message)
          (recur (read-user-input!) messages-including-response))))))
```

# Demo
> **You**: Howdy, can you tell how the weather in Paris is like?
>
> **LLM**: Howdy — right now it's showing -26 °C in Paris (France), which is about -14.8 °F. That’s extremely cold and unusual for Paris — would you like me to:
> - confirm you meant Paris, France (not another Paris),
> - get a short-term forecast or current conditions (wind, precipitation), or
> - convert to a different unit?
>
> **You**: quit

As you can see, the tool returns a fixed "-26 C" for any query. The LLM accepts the tool's output and comments on how unusual the temperature is in Paris. If you ask it for the weather in a different city which returns the same output, the model concludes that something is wrong with the `get_current_weather` tool and asks you to double check.

That's it for this post. In the next one, let's look at how to remove all the hardcoding in the tool description to the LLM and utilize some of Clojure's metadata functions to generate them.
