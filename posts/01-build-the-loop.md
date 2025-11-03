Title: Building a Coding Agent : Part 1 - A basic LLM chat loop
Date: 2025-10-29
Tags: Clojure, OpenAI, LLM

I am in the process of learning Clojure and wanted a small project to pursue.
Given the whole AI Agents hype came across a nice post by **Thorsten Ball** on building a code editing agent. He utilizes Go and an Anthropic model endpoint to show how easy it is to build a code editing agent. You can read the whole post at [How to Build an Agent or: The Emperor Has No Clothes](https://ampcode.com/how-to-build-an-agent)

I thought I would try and replicate the same process using Clojure and an OpenAI model (gpt-5-mini).

# How chat completions work?
Before we get to implementing a code-editing agent, let's spend some time understanding how an LLM based chat workflow works. An LLM is a next word (token) prediction engine and it relies on the previous tokens to predict the next word. So, for a chat based experience to work it effectively needs the entire conversation history to generate the next response. This is due to the fact that it is stateless and all the state of the conversation is in the history. It is the client which talks to the LLM which maintains the history of the conversation.

![LLM Completion](assets/llm-loop.png)

Each message in the conversation history is tagged with a role attribute - **user** or **assistant**

# OpenAI API Client

For talking to the OpenAI model we will use the [openai-clojure](https://github.com/wkok/openai-clojure) library. Add it to your `deps.edn` file as
```clojure
{:deps {org.clojure/clojure {:mvn/version "1.12.0"}
        net.clojars.wkok/openai-clojure {:mvn/version "0.23.0"}}
 :paths ["src" "test"]}
```

The library supports both OpenAI and Azure hosted OpenAI models.
The chat-completion API takes a model parameter which can be passed in.

```clojure
(openai/create-chat-completion {:model (:model config)
                                :messages messages}
                               (select-keys config
                                [:api-key :api-endpoint :impl]))
```

The `api-keys` and `api-endpoint` parameters need to be pointed to your instance of the model. The `:model` parameter is the model we are using. In my case it is the `gpt-5-mini` model.
The `:messages` parameter contains the conversation history. The last message in the history is typically a **user** message.

# Code for the loop

Now that we know how to call the API. Let's look at the main loop of the agent. Right now it can only respond via text and does not do much but can still be used to ask questions like with any LLM Chat interface.

```clojure
(defn -main
  []
  (let [config (read-config)]
    (loop [user-message (get-user-message)
           messages []]
      (when (some? user-message)
        (let [new-messages (add-message-to-history messages user-message :user)
              response (-> new-messages
                           (call-llm-api config)
                           extract-first-response)
              history (add-message-to-history new-messages (:content response) :assistant)]
          (print-llm-response response)
          (recur (get-user-message) history))))))
```

That is all that is needed to make the LLM respond to your messages. This small 12 line function. We read the user input in the `get-user-message` function. Append it to the history and invoke the chat completion API using the `call-llm-api` call. Once we get the response we add the response to the history and print it out for the user to see. We restart the loop all over to collect the next user input.

Here is the full listing of the code. The code is simplified a bit to ignore multiple responses from the LLM.
```clojure
(ns agent.core
  (:require
   [clojure.edn :as edn]
   [wkok.openai-clojure.api :as openai]))

(defn get-user-message
  []
  (print "User => ")
  (flush)
  (let [message (read-line)]
    (when-not (= message "quit")
      message)))

(defn print-llm-response
  [message]
  (println "LLM =>" (:content message)))

(defn call-llm-api
  "Call the chat completion API"
  [messages config]
  (openai/create-chat-completion {:model (:model config)
                                  :messages messages}
                                 (select-keys config [:api-key :api-endpoint :impl])))

(defn extract-first-response
  "The LLM call can return multiple responses. Extract the first one and use it"
  [response]
  (let [choices (:choices response)
        responses (map #(get-in % [:message]) choices)]
    (when-not (= 1 (count responses))
      (println "Expected exactly one response" {:responses responses}))
    (first responses)))

(defn add-message-to-history
  "Adds a message with the specified role to the message history."
  [messages message role]
  (conj messages {:role role :content message}))

(defn read-config
  []
  (edn/read-string (slurp "llm.edn")))

(defn -main
  []
  (let [config (read-config)]
    (loop [user-message (get-user-message)
           messages []]
      (when (some? user-message)
        (let [new-messages (add-message-to-history messages user-message "user")
              response (-> (call-llm-api new-messages config)
                           extract-first-response)
              history (add-message-to-history new-messages (:content response) "assistant")]
          (print-llm-response response)
          (recur (get-user-message) history))))))
```

The `llm.edn` config file is of the format:
```clojure
{:api-key "REPLACE-YOUR-KEY"
 :api-endpoint "https://<ENDPOINT>.openai.azure.com"
 :impl :azure
 :model "gpt-5-mini"}
```
You can also export the API key as an environment variable to be picked from. Have a look at the [openai-clojure](https://github.com/wkok/openai-clojure) docs for more details.
# Demo of the loop
Here is a sample run, asking for the weather in Paris. As you can see the LLM complains of having no access to live data for the weather however the model has been provided information about the current date already by whatever **system** prompt that has been initialized by the model provider. So, it can predict how the weather will be in October. In the next post we will look at adding a weather tool which the model can call to get the weather in a particular location.

>
> **LLM** => Howdy! How can I help you today?
>
> **User** => How is the weather in Paris today?
>
> **LLM** => I don’t have access to live weather data, so I can’t tell you the exact conditions in Paris right now. I can, however:
>
> - Give a typical late‑October picture (expect cool, often cloudy, with a good chance of rain; daytime highs commonly around 10–15 °C / 50–59 °F and nights near 5–9 °C / 41–48 °F), or
> - Walk you through quick ways to get the current weather (Google “weather Paris”, check MeteoFrance, Weather.com, or OpenWeatherMap, or ask your phone assistant), or
> - If you paste a current weather webpage or let me know a specific source, I can help interpret it.
>
> Which would you like? (Also: Celsius or Fahrenheit?)
>
> **User** => quit
```
