# Task specifications

This directory defines the 53 MeMoREval tasks and the four stacked composite tasks.
Each task follows the experience–request protocol from Section 4.1 of the paper.

```
tasks/
  index.csv            one row per task: scene, category, request counts, composite membership
  office/  studio/  home/  clinic/
    <task>.json        a single task
    stacked_<scene>.json  the scene's stacked composite task
```

| Scene  | Tasks | In stacked composite |
|--------|------:|---------------------:|
| office | 12 | 10 |
| studio | 14 | 10 |
| home   | 14 | 10 |
| clinic | 13 | 11 |
| **all** | **53** | **41** |

`index.csv` matches Table 8 of the paper.

## Task file

```jsonc
{
  "schema_version": "1.0",
  "task": "pref_water_carafe",
  "scene": "office",
  "category": "user_preference",
  "in_stacked_composite": true,
  "experience": [ /* experience segments, in order */ ],
  "requests":   [ /* information requests first, then action requests */ ]
}
```

`category` is one of the seven categories in Table 1 of the paper.
They are `object_location`, `spatial_configuration`, `appearance_binding`, `set_aggregation`, `user_preference`, `occluded_state` and `skill_outcome`.

### Experience

The robot executes the experience phase as a fixed script.
The memory module records the resulting observations without knowing the requests.
The first segment is always the scene tour, which visits every room.
Later segments add task events such as a moved object, an opened container or a user message.

| Field | Meaning |
|---|---|
| `id` | Segment identifier. The tour is `<scene>_tour`. |
| `instruction` | Instruction shown for the segment. |
| `user_messages` | User utterances delivered during the segment, if any. |
| `scene_changes` | Changes applied to the simulator before the segment starts, if any. |
| `skill_settings` | Skill configuration for the segment, if any. |
| `skill_calls` | The scripted skill calls the robot executes. |

### Requests

Each request starts from a fresh agent context containing only the request and the robot's current room.
The agent receives no new observations and can use only its memory.

**Information requests** have `"type": "information"`.

| Field | Meaning |
|---|---|
| `instruction` | The text given to the agent. |
| `question` | The question alone. |
| `answer_type` | `region`, `color`, `category`, `count`, `set` or `side`. |
| `reference_answer` | The graded answer. |
| `grader` | How the submitted answer is compared, described below. |
| `allowed_skills` | The skills the agent may call. Information requests allow only `answer`. |
| `max_steps` | Step budget for the request. |

**Action requests** have `"type": "action"`.
The agent submits a complete plan of skill calls.
The plan is executed open-loop, with no observations returned and no chance to revise it.

| Field | Meaning |
|---|---|
| `instruction` | The text given to the agent. |
| `goal` | Goal type, one of `fetch`, `gather`, `restore` or `deliver`, with an object hint. |
| `success` | The predicate checked on the final simulator state and skill trace, described below. |
| `reference_plan` | A plan that satisfies `success`. It shows that the request is solvable and is never shown to the agent. |
| `scene_changes` | Changes applied to the simulator before the request, if any. |
| `skill_settings` | Skill configuration for the request, if any. |
| `max_steps` | Step budget for the request. |

### Stacked composites

`stacked_<scene>.json` concatenates the experience of its `components` into one longer history.
It then issues every component's information requests, followed by every component's action requests.
Each request carries a `component_task` field and keeps its original grading.
Before each action request the simulator is restored to its state at the end of the combined experience.
A component task passes when all of its requests pass.

## Skills

| Skill | Arguments | Notes |
|---|---|---|
| `navigate_to` | `region` or `target` | Go to a room, or to an object or piece of furniture. |
| `pick` | `target` | Grasp an object. |
| `place` | `on` or `in` | Place the held object on or in a receptacle. |
| `place_a`, `place_b` | `dest` | Two placement variants used by skill-outcome tasks. One of them fails, as set by `skill_settings.failing_place_variant`. |
| `open`, `close` | `target`, `subtarget` | Open or close an articulated part, such as a door or drawer of a cabinet. |
| `answer` | | Respond to an information request. |

Object and furniture identifiers, such as `water_carafe_0`, refer to instances in the scene files.

## Scene changes

| `op` | Fields | Effect |
|---|---|---|
| `move_object` | `id`, and `region` or `receptacle` | Move an object without the robot acting. |
| `repark_robot` | `region` | Move the robot to a region. |

## Graders for information requests

Answers are normalized for case, whitespace and separators before comparison.
For example, `water_carafe`, `water carafe` and `Water-Carafe` are the same answer.

| Grader | Passes when |
|---|---|
| `exact` | The normalized answer equals the reference. |
| `numeric` | The integer answer equals the reference. |
| `set_match` | The submitted list matches the reference list one to one. An added leading modifier is tolerated, so `glass water carafe` matches `water_carafe`. |
| `set_match_any` | The submission passes `set_match` against any one of several alternative references. |

## Success predicates for action requests

| Predicate | Passes when |
|---|---|
| `fetched_onto` | The agent successfully picked `instance` and it rests on `target`. |
| `object_in_region` | `instance` is in `region`. |
| `all_in_region` | Every one of `instances` is in `region` and none is held. |
| `first_place_skill` | After successfully picking `instance`, the first placement aimed at `destinations` uses the placement variant `skill`. |
| `first_place_key` | After successfully picking `instance`, the first placement aimed at `destinations` uses the argument `key`, such as `in` rather than `on`. |
| `any_of` | Any one of the child `criteria` passes. |

The two `first_place_*` predicates grade the first choice rather than the final state.
An agent that forgot the working option could otherwise fail, retry, and still reach the same final state.

## Protocol limits used in the paper

- The agent may make at most five retrieval calls before responding to a request.
- At most 20,000 characters of textual memory are rendered into the agent's context per turn.
- Retrieved frames, descriptions and scene-graph records do not count toward that limit.
