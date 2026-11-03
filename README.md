# retico-cozmoAgent
A ReTiCo module for a voice-controlled Cozmo robot agent, built with [SmolAgents](https://github.com/huggingface/smolagents). It turns natural speech commands into step-by-step plans of Cozmo robot behaviors.

The module doesn't control the robot itself. Instead, it produces a `GenericDictIU` for each step of the plan, which you can pass to a dialogue manager or a Cozmo module to make Cozmo carry out the task.

A video tutorial can be found [here](https://drive.google.com/file/d/1ObVg8ZZqoUvDrD-2CKwEPTUWRmtsg_9d/view?usp=drive_link).

## Installation and Requirements

To use the Cozmo agent module you first need to install the retico-cozmoagent package:
* Install retico-cozmoagent:
```pip install retico-cozmoagent```

  or, for the latest development version from GitHub:
```pip install git+https://github.com/retico-team/retico-cozmoAgent.git```

This also installs its Python dependencies (`retico-core` and `smolagents` 1.19). Python 3.10 or newer is required.

### API key
The agent uses an LLM through an OpenAI-compatible API. By default it uses DeepSeek V3 (`deepseek/deepseek-chat-v3-0324:free`) through [OpenRouter](https://openrouter.ai/). Either pass your API key as the `api_key` argument, or set the `OPENROUTER_API_KEY` environment variable:
```bash
OPENROUTER_API_KEY=<your-api-key>
```

## Modules

### `CozmoSmolAgentsModule`
Subscribes to `SpeechRecognitionIU` (e.g. from `WhisperASRModule`) and outputs `GenericDictIU`. Once the user's speech is committed, the agent turns it into an execution plan and sends one committed `GenericDictIU` per step. If the command is unclear, the agent asks for clarification with `say_text()` instead. The last 8 messages of the conversation are given to the agent as context.

Each `GenericDictIU` payload has the form:
```python
{
    'function': 'drive_straight',          # the Cozmo behavior to run
    'parameters': {'distance_mm': 200, 'speed_mmps': 50},
    'step_number': 1,                       # the position of this step in the plan
    'tool_result': {...}                    # the raw output of the agent's tool call
}
```

Available Cozmo behaviors:
* `drive_straight`: drive forward or backward (`distance_mm`, `speed_mmps`)
* `turn_in_place`: turn left (positive) or right (negative) (`angle_degrees`)
* `say_text`: make Cozmo speak (`text`)
* `move_head`: lift (positive) or lower (negative) the head (`radians`)
* `look_around`: look around for a cube
* `go_to_object`: find a cube and drive up to it (`distance_mm`)
* `drive_to_charger`: drive to the charger

#### Arguments:
* `model_id` (str): the model the agent uses; defaults to `"deepseek/deepseek-chat-v3-0324:free"`
* `api_base` (str): the URL of the OpenAI-compatible server; defaults to `"https://openrouter.ai/api/v1"`
* `api_key` (str): the API key for the server; defaults to `None` (uses `OPENROUTER_API_KEY`)

### Example
**Note:** The example uses `MicrophoneModule` and `DebugModule` from `retico_core` and `WhisperASRModule` from [retico-whisperasr](https://github.com/retico-team/retico-whisperasr) (`pip install retico-whisperasr`).

```python
from retico_core.debug import DebugModule
from retico_core.audio import MicrophoneModule
from retico_whisperasr.whisperasr import WhisperASRModule
from retico_cozmoagent import CozmoSmolAgentsModule

mic = MicrophoneModule(chunk_size=320, rate=16000)
debug = DebugModule()
asr = WhisperASRModule()
dm = CozmoSmolAgentsModule()

mic.subscribe(asr)
asr.subscribe(dm)
dm.subscribe(debug)

mic.run()
asr.run()
dm.run()
debug.run()

input()

asr.stop()
mic.stop()
dm.stop()
debug.stop()
```