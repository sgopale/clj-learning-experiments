Title:  Building a Coding Agent : Part 9 - Adding Command Handling
Date: 2025-11-19
Tags: Clojure, OpenAI, LLM
If we look at the code for the agent right now, the chat loop is messy. It looks like
```
Get the user input -> Invoke the LLM
```
The code for the main loop looks like right now.
```clojure
(loop [user-message (read-user-input!)
           messages [(get-system-prompt)]]
      (when (some? user-message)
        (let [new-messages (add-message-to-history messages user-message)
              {:keys [history usage]} (get-assistant-response new-messages config mcp-tools combined-registry)
              assistant-message (:content (last history))]
          (display-assistant-response! assistant-message)
          (dbg-print usage)
          (recur (read-user-input!) history))))
```

There is no way currently to do other things besides quit. That is also possible because we had added a hard-coded check inside the `read-user-input!` function to see if the user entered a `quit` message. When the user enters `quit` we return an empty input the same as if the user enter a EOF via Ctrl+D. This terminates the chat loop which is checking for the presence of some input from the user.

If you look at other agents available they provide /commands which allow the user to modify things like the conversation history, change models and so on. However, with our current read-user-input! method none of that is possible.

Let us refactor the method to make it extensible easily. We will achieve this by converting the simple loop into a state transition loop. We will add a new function `handle-user-input!` which will process special commands and indicate a state transition via a next state return value. Also, we want to be able to achieve things like clearing conversation history and changing models, so the `handle-user-input!` function needs to be able to change the state. To achieve this we will create a state map which is consists of the following:
```clojure
{
  :history [] ; A vector which holds the conversation history
  :prompts {} ; A map which holds default prompts like the system prompt
  :config {} ; A map containing the model information
  :tools [] ; A vector of tools available to the model. Either MCP or coded tools
  :tool-registry {} ; A map of tool name to the invocation function. This will allow us to handle tool calls from the model
  :next-state :key ; The next state which the LLM chat loop should transition to
}
```
In this version we will support three states:
- :quit Quit the app
- :llm Invoke the LLM API with the current history
- :user Get input from the user

With these three states available our chat loop becomes simpler. The `handle-user-input!` function takes in the current state and returns a new state. This makes it easy for us to implement commands like clearing history, changing the model etc. As we can change the configuration which is stored inside the state map.
After the implementation of our `handle-user-input!` function the main loop looks like:
```clojure
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
        (recur (state/handle-user-input! current-state (read-user-input!))))
```
Our `handle-user-input!` function can be written as:
```clojure
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
With this function it is trivial to add new commands. I have added commands for quitting, clearing history, generating debug output and switching models. This is much better than the old loop which could only handle quit commands. This will set us up for adding more commands like saving and loading conversations as well. I think this kind of clean state pattern is easily achievable in Clojure which forces immutability on the programmer. If I had used a different programming language which allowed mutation easily, I would have state changes all over the code. This also makes it very easy to test this code as well as the inputs and outputs are predictable.

The full listing of the code is [here](09-full-listing.html)
