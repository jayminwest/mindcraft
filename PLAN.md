# Mindcraft Cognitive Enhancement Project - Implementation Plan

This document outlines the steps to modify the Mindcraft codebase to achieve the objectives of the Cognitive Enhancement Project.

## 1. Implement Enhanced Memory (STM/LTM)

**Goal:** Introduce structured Short-Term Memory (STM) and a mechanism for Long-Term Memory (LTM) consolidation/summarization.

**Files to Modify:**

*   **`src/agent/history.js`**:
    *   **STM:** Decide on the STM structure. Potentially keep `this.turns` as STM but manage its size more explicitly (e.g., using `settings.max_messages` as a hard limit). **Ensure the chosen structure is JSON-serializable.**
    *   **LTM Consolidation:** Refactor the `summarizeMemories` method. Instead of just creating a single `this.memory` string, implement logic to generate summaries of STM chunks and append them to a structured LTM (which could still be stored in `this.memory` or a new property). Consider the frequency and trigger for consolidation. **Ensure the structured LTM is JSON-serializable.**
    *   **Persistence:** Update `save()` and `load()` to correctly handle JSON serialization and deserialization of the new STM/LTM structures into/from `memory.json`.
*   **`src/models/prompter.js`**:
    *   **Prompt Injection:** Modify `replaceStrings`. Introduce distinct placeholders (e.g., `$STM`, `$LTM`) and update the function to populate them with the relevant memory content from `agent.history`. Update profile prompts (`conversing`, `coding`, etc.) to use these new placeholders instead of just `$MEMORY`.
*   **`src/agent/memory_bank.js`**:
    *   **Review/Refactor:** This currently seems like a simple wrapper. Evaluate if it's still needed or if its functionality should be merged into `history.js` after the memory enhancements.

## 2. Introduce Internal State & Needs

**Goal:** Enable agents to track internal states (health, hunger) and basic needs, influencing their behavior.

**Files to Modify:**

*   **`src/agent/agent.js`**:
    *   **State Properties:** Add new properties to the `Agent` class (e.g., `this.internalState = { health: 20, food: 20, saturation: 5 }`). **Note: This simple object structure is inherently JSON-compatible.**
    *   **State Update:** Implement a method (e.g., `_updateInternalState()`) called periodically by the main `update()` method. This new method should read values from `this.bot.health`, `this.bot.food`, `this.bot.foodSaturation` and update `this.internalState`.
    *   **Persistence (Optional):** If desired, modify `history.save()` and `history.load()` (via `agent.js`) to persist `this.internalState` using JSON serialization.
*   **`src/models/prompter.js`**:
    *   **Prompt Injection:** Modify `replaceStrings` to add a new placeholder (e.g., `$INTERNAL_STATE`) and populate it with a formatted string representing the agent's current health, hunger, etc., from `agent.internalState`. Update relevant profile prompts.

## 3. Develop Proactive Thinking (Cognitive Cycle)

**Goal:** Implement a simple cognitive cycle allowing agents to formulate goals and initiate actions based on their internal state and memory, independent of direct user commands.

**Files to Modify:**

*   **`src/agent/self_prompter.js`**:
    *   **Cognitive Cycle Logic:** Enhance the `update()` or `startLoop()` method. Before generating the self-prompt message (`You are self-prompting with the goal...`), add logic to:
        *   Read `agent.internalState` (from Goal 2).
        *   Read recent STM from `agent.history` (from Goal 1).
        *   Evaluate needs: If hunger is low, set an internal goal like `"Find food"`. If health is low, set `"Seek safety and regenerate"`.
        *   Evaluate memory: If STM contains actionable information (e.g., "User mentioned a nearby cave"), set an internal goal like `"Explore the cave mentioned by the user"`.
        *   Prioritize needs/goals.
        *   Update `this.prompt` with the highest priority internal goal *before* calling `agent.handleMessage`.
    *   **State Integration:** Ensure the self-prompter checks `this.state` (ACTIVE, PAUSED, STOPPED) correctly in conjunction with the agent's idle status (`agent.isIdle()`) and internal state needs.
*   **`src/agent/agent.js`**:
    *   **Integration:** Verify that `agent.update()` correctly calls `self_prompter.update()` to drive the cycle.

## 4. Implement Robust LLM Tool Use (Structured JSON)

**Goal:** Refactor action execution to use LLM-generated JSON specifying predefined tools, replacing direct code generation.

**Steps & Files to Modify:**

1.  **Define Tool Schema:**
    *   Decide on a standard JSON format. Example:
        ```json
        {
          "tool": "<tool_name>",
          "args": {
            "<arg_name_1>": "<value_1>",
            "<arg_name_2>": <value_2>
          }
        }
        ```
2.  **Define Core Tools:**
    *   Identify essential functions in `src/agent/library/skills.js` and `src/agent/library/world.js` to expose. Examples: `craftRecipe`, `goToPosition`, `collectBlock`, `attackNearest`, `placeBlock`, `getInventoryCounts`, `getNearestBlock`, `smeltItem`, `equip`, `discard`.
    *   Document the exact arguments (name, type, description) for each tool.
3.  **Modify Prompts:**
    *   **`src/models/prompter.js`**:
        *   Update system prompts in profiles (`this.profile.conversing`, potentially others) accessed via `promptConvo`. Clearly instruct the LLM to respond *only* with JSON matching the defined schema when an action is required.
        *   Provide the list of available tools and their arguments within the prompt context (potentially using a new placeholder like `$AVAILABLE_TOOLS`).
        *   Update `replaceStrings` to populate `$AVAILABLE_TOOLS`.
    *   **`src/models/<specific_model>.js` (e.g., `gemini.js`)**:
        *   Ensure the `sendRequest` method is compatible with receiving JSON responses. Some models might have specific modes or parameters for JSON output.
4.  **Refactor Action Execution:**
    *   **`src/agent/action_manager.js`**:
        *   **Core Change:** Rewrite the logic within `_executeAction` (or potentially `runAction`). It should no longer expect `actionFn` to be executable code.
        *   **JSON Parsing:** Receive the LLM's response string. Attempt to parse it as JSON. Handle parsing errors.
        *   **Validation:**
            *   Check if the parsed object contains the required `tool` and `args` keys.
            *   Validate `tool` against the list of defined tools.
            *   Validate the keys and types of values within `args` against the expected parameters for the chosen tool. (Leverage `src/utils/mcdata.js` for block/item name validation if needed. Consider adapting validation logic from `src/agent/commands/index.js`).
        *   **Function Mapping:** Create a map or use conditional logic to associate the validated `tool` string with the actual function reference (e.g., `skills.craftRecipe`, `world.goToPosition`).
        *   **Safe Execution:** Call the mapped function using `await mappedFunction(this.agent.bot, ...Object.values(validated_args));`. Use a `try...catch` block to handle runtime errors from the skill/world functions.
        *   **Return Value:** Update the return structure (`{ success, message, interrupted, timedout }`) based on the outcome of the tool execution.
    *   **`src/agent/agent.js`**:
        *   **Response Handling:** Modify `handleMessage`. When processing an LLM response (`res`), instead of checking for `!commandName` with `containsCommand`, check if `res` is a JSON string representing a tool call.
        *   If it's JSON, pass the raw JSON string (or the parsed object) to `this.actions.runAction` (or the refactored execution logic).
        *   If it's not JSON, treat it as a conversational response as before.
5.  **Review/Remove Code Execution:**
    *   **`src/agent/coder.js`**: Evaluate if this is still needed. If the goal is *only* structured tool use, this file and its associated templates/logic can likely be removed or significantly simplified. If complex, multi-step plans are desired, it might be adapted to generate sequences of tool calls instead of raw JS.
    *   **`bots/execTemplate.js`**: Likely becomes obsolete if `coder.js` is removed.
    *   **`settings.js`**: Remove or disable `"allow_insecure_coding"`.
6.  **Command System (`src/agent/commands/index.js`)**:
    *   Decide the future of the `!` command syntax. It could be:
        *   Removed entirely.
        *   Kept for direct user control/debugging, bypassing the LLM.
        *   Adapted so `executeCommand` internally calls the new `action_manager` logic.
    *   Reuse argument validation logic if applicable for the JSON tool arguments.

## 5. Implement Persistence for Enhanced State

**Goal:** Ensure new memory structures and internal states are saved and loaded correctly.

**Files to Modify:**

*   **`src/agent/history.js`**:
    *   Verify `save()` correctly serializes the enhanced STM/LTM structures (from Goal 1) into a JSON-compatible `data` object.
    *   Verify `load()` correctly deserializes and restores STM/LTM from the loaded JSON `data`.
*   **`src/agent/agent.js`**:
    *   If persisting internal state (from Goal 2) is desired:
        *   Modify `history.save()` callsite (or add logic within `history.save` itself) to include the JSON-compatible `this.internalState` in the saved data.
        *   Modify the `load_mem` handling in `agent.start()` to restore `this.internalState` from the loaded JSON data.

## 6. Adapt for Multi-Agent Scaling

**Goal:** Ensure enhancements work effectively in multi-agent scenarios.

**Files to Modify/Review:**

*   **`src/agent/conversation.js`**:
    *   Review how the cognitive cycle (Goal 3) interacts with conversation handling. Does self-prompting interrupt or delay responses inappropriately? Adjust logic in `_scheduleProcessInMessage` or `SelfPrompter` if needed.
*   **`src/agent/self_prompter.js`**:
    *   Consider if self-prompting needs awareness of other agents' actions or states, potentially accessed via `conversation.js` or shared state mechanisms (if implemented).
*   **`src/agent/action_manager.js`**:
    *   Ensure the new tool execution logic is robust against potential multi-agent conflicts (e.g., two agents trying to mine the same block simultaneously - though Mineflayer might handle some of this). Explicit locking or coordination might be needed for complex shared tasks, but is likely beyond initial scope.
*   **`src/server/mind_server.js`**:
    *   No immediate changes likely needed, but it remains the central point for observing multi-agent interactions.

**General Approach:**

*   Implement goals sequentially or in parallel where feasible (e.g., Memory and State can be done somewhat independently before integrating into the Cognitive Cycle).
*   Goal 4 (Tool Use Refactor) is the most significant architectural change and should be planned carefully.
*   Test thoroughly after each major change, especially the tool use refactor.
*   Use version control (git) extensively. Create branches for each major feature.
