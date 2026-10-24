# ShelfLife: Four-Person Hackathon Build Plan

## Product goal

Help café staff monitor a produce storage area, identify visible changes, review image evidence, and request removal of a flagged item. Monitoring is the main dashboard feature. A digital twin demonstrates the operator-approved removal.

**Demo story:** A camera observes a fixed produce tray. An inspection flags increased banana browning. An operator selects the finding, compares images, and clicks **Remove item**. The virtual robot moves that item into a reject bin, then the inventory updates.

## Scope and simple stack

| Area | Technology | Weekend scope |
| --- | --- | --- |
| Backend | Python, FastAPI, Uvicorn | Camera capture, inspection API, state, removal requests |
| Frontend | React, HTML, CSS, JavaScript | Camera-first dashboard, review queue, evidence, actions |
| Digital twin | Three.js, URDFLoader, ROS 2 Python node | Existing robot URDF and predefined removal trajectories |
| Computer vision | Python, OpenCV | Fixed slots, one fruit type, visible browning comparison |

Run locally for the judging demo. Avoid cloud infrastructure, a database service, training a large model, and physical robot control for the MVP. Use a webcam, controlled lighting, and a tray with clearly separated slots.

The current published dashboard is a UI prototype: its findings are presets and its robot is an illustrative animation. The work below replaces those pieces with connected functionality.

## Role 1: Backend and integration owner

**Own the path from camera observations to shared application state.**

### Responsibilities

- Create the FastAPI application and launch it with Uvicorn.
- Own the camera connection and capture loop. Open the webcam only once; expose the same frames to CV and the browser.
- Serve an MJPEG stream or periodically refreshed JPEG frames to the dashboard.
- Call the CV module on a captured frame when an inspection is requested. Start with manual inspections; add periodic checks after the full flow works.
- Store current slot observations, review decisions, and removal states. An in-memory store plus a small JSON event file is sufficient.
- Validate removal requests: item exists, is not already removed, and has no active removal.
- Bridge removal requests into ROS 2 and consume completion/failure messages.
- Update inventory only after successful simulated removal. A failed or disconnected command must not remove inventory.
- Provide sample responses immediately so frontend and twin work can proceed independently.

### Deliverables

- Working API and camera stream.
- Shared request/response examples.
- ROS bridge and removal state transitions.
- Setup instructions and an explicit demo replay mode if the camera or ROS is unavailable.

### Done when

A browser can receive a frame, run an inspection, request one removal, and observe its completion. Duplicate requests do not start duplicate motions.

## Role 2: Frontend and monitoring dashboard owner

**Own the operator experience, with monitoring as the primary view.**

### Responsibilities

- Build the large storage-room camera panel, camera status, and last-frame timestamp.
- Overlay selectable slot regions using CV results and normalized image coordinates.
- Show a review queue with slot identity, observation time, and a plain-language finding.
- Display baseline and current image crops together, including a measured change and any visibility limitations.
- Support **Inspect now**, **Mark inspected / keep**, and **Remove item** actions.
- Open the digital twin when removal starts; allow returning to the camera monitor.
- Show requested, executing, completed, failed, and disconnected states. Disable duplicate removal controls while a command is active.
- Clear stale detections or show a stale-feed warning when frames stop arriving.
- Clearly distinguish live video, recorded replay, and simulated removal.

### Deliverables

- Camera-first React dashboard.
- API client and shared selected-slot state.
- Embedded twin component and activity log.
- Responsive layout suitable for a judging laptop.

### Done when

An operator can follow the entire workflow without developer tools, and every finding is linked to the correct visible slot and evidence.

## Role 3: Digital twin and ROS owner

**Own the virtual removal demonstration and its status messages.**

### Responsibilities

- Load the existing robot URDF and meshes in Three.js with URDFLoader.
- Add a tray, named item slots, and a reject bin. Match slot IDs to the camera/CV configuration.
- Define the camera-slot-to-twin mapping manually for the MVP. This is a configured mapping, not automatic 3D reconstruction.
- Prepare one reachable pick-and-place trajectory for each supported slot. Use existing tools or hand-verified joint waypoints; avoid building a general motion planner.
- Write a ROS 2 subscriber for removal requests and publish simulated joint states.
- Forward joint updates through the backend so the browser animates the same trajectory.
- Attach the virtual item to the gripper at pickup, then release it into the reject bin.
- Publish completion only after the full animation sequence finishes; publish failure for unknown slots or invalid commands.
- Label motion as simulation. Do not send commands to a physical robot.

### Deliverables

- Reusable twin viewer component.
- Robot assets, slot map, and predefined trajectories.
- ROS simulation node and documented topics.
- Standalone test command to remove one virtual item.

### Done when

A request for slot A2 moves only A2, the correct robot joints animate, and the backend receives the matching completion event.

## Role 4: Computer vision and evaluation owner

**Own a narrow, explainable visible-change detector.**

A large learned model is not required. Start with an OpenCV inspection pipeline that your team can demonstrate and measure reliably.

### Responsibilities

- Use one supported fruit type, ideally bananas, in fixed labeled slots under a fixed camera.
- Define each slot's image region and capture a baseline image.
- Segment the visible fruit from the background using controlled tray colors and color/contour rules.
- Measure visible brown-region proportion and compare it with the baseline.
- Return the measured change, image crop, slot identity, and assessment status.
- Reject or flag frames with substantial occlusion, changed camera alignment, or poor lighting. Do not turn a bad image into a confident finding.
- Create a small evaluation set with good lighting, changed lighting, an empty slot, partial occlusion, and visibly browned fruit.
- Report false alerts and missed changes on those examples; keep thresholds configurable.
- Prepare a clearly labeled recorded time sequence for deterioration over days, since it will not occur during judging.

### Deliverables

- A Python inspection function with no web or ROS dependency.
- Baseline/current evidence crops and optional region masks.
- Slot configuration and documented thresholds.
- Small evaluation summary and replay images.

### Done when

The detector flags a visible change on supported examples, handles an empty or obscured slot explicitly, and returns the schema agreed with the backend.

**Claim boundary:** Detect visible quality changes and recommend review. Do not claim to certify food safety or infer hidden spoilage.

## Shared interfaces: agree before coding

### REST endpoints

| Endpoint | Purpose | Owner |
| --- | --- | --- |
| `GET /camera/stream` | Browser camera stream | Backend |
| `GET /state` | Slots, findings, feed status, current removal | Backend |
| `POST /inspect` | Inspect a captured frame | Backend + CV |
| `POST /slots/{slot_id}/review` | Record keep/review decision | Backend |
| `POST /removals` | Request simulated removal and return command ID | Backend + twin |
| `GET /removals/{command_id}` | Poll removal progress | Backend |
| `GET /twin/joints` | Latest joint names and positions | Backend + twin |

Use simple polling first. Add a WebSocket for joint updates only if polling visibly limits animation quality. Keep any network client isolated so transport can change without rewriting the viewer.

### CV function contract

```python
inspect_frame(frame, baseline, slot_config) -> list[dict]
```

Example slot observation; numbers are illustrative:

```json
{
  "slot_id": "A2",
  "item_type": "banana",
  "observed_at": "2026-10-24T21:00:00Z",
  "assessment": "review",
  "reason": "Visible browning increased",
  "brown_fraction": 0.34,
  "baseline_brown_fraction": 0.08,
  "change_percentage_points": 26,
  "bbox_normalized": [0.40, 0.31, 0.18, 0.20],
  "baseline_image_url": "/evidence/A2/baseline.jpg",
  "current_image_url": "/evidence/A2/current.jpg",
  "mode": "live"
}
```

`bbox_normalized` is `[x, y, width, height]`, relative to the full source image. Frontend overlays must account for letterboxing. `assessment` is `monitor`, `review`, `unassessable`, or `empty`. Do not report a confidence percentage unless it has a defensible meaning and validation.

### Removal contract

```json
{
  "command_id": "remove-001",
  "slot_id": "A2",
  "mode": "simulation",
  "status": "requested"
}
```

State sequence: `requested → executing → completed`, or `failed`. Backend owns the authoritative status; the frontend displays it. Keep the command ID on every event so late messages cannot complete the wrong request.

### ROS topics

| Topic | Message | Purpose |
| --- | --- | --- |
| `/shelflife/remove_request` | `std_msgs/String` containing the removal JSON | Trigger a predefined simulated trajectory |
| `/joint_states` | `sensor_msgs/JointState` | Drive the robot URDF joints; positions in radians |
| `/shelflife/removal_status` | `std_msgs/String` containing command ID, slot ID, status, optional error | Report execution outcome |

Camera capture, CV, and dashboard state use ordinary Python/HTTP. ROS is limited to robot simulation and its bridge.

## Suggested repository ownership

```text
shelflife/
  frontend/       # Frontend owner
  backend/        # Backend owner
  vision/         # CV owner
  twin/           # Twin owner: viewer, assets, slot map
  ros_ws/src/     # Twin owner: ROS simulator
  contracts/      # Shared JSON examples and schemas
  demo_data/      # CV owner: labeled replay frames
  README.md       # Backend owner coordinates setup
```

Keep the CV package independent of FastAPI and the twin viewer independent of the dashboard layout. Avoid multiple people editing the same component at once.

## Weekend milestones

| Stage | Backend | Frontend | Twin | CV |
| --- | --- | --- | --- | --- |
| First 1–2 hours | Define API and return mock state | Build camera-first layout against mocks | Load URDF and one slot | Capture images and define slot regions |
| First working slice | Serve camera frames and one real result | Render feed, overlay, and evidence | Animate one removal | Detect browning for one slot |
| Integration checkpoint | Connect removal request/status | Connect queue and removal controls | Consume request, publish joints/status | Return full agreed result format |
| Expansion | Handle errors and record events | Show stale/failed/empty states | Add remaining fixed slots | Test lighting and occlusion |
| Final 3–4 hours | Freeze API and rehearse startup | Polish readability and demo flow | Verify repeatable reset | Prepare labeled replay and evaluation |

**Integrate the first real end-to-end flow before expanding the number of items.**

## Minimum successful judging demo

1. Show a live storage-area camera feed with fixed produce slots.
2. Run inspection and select a flagged slot.
3. Show baseline/current evidence and the measured visible change.
4. Click **Remove item** to start the virtual robot sequence.
5. Show completed status, updated inventory, and the activity record.
6. Reset and repeat reliably.

Use a real browned fruit for a live inspection or explicitly switch to recorded replay. Never present a prerecorded deterioration sequence as live time progression.

## Cut these first if time is tight

- Arbitrary storage-room object recognition.
- Multiple fruit types and overlapping piles.
- Automatic camera-to-robot calibration.
- General grasp planning, collision avoidance, or physical robot execution.
- Cloud hosting of the local ROS stack.
- Recipes, purchasing analytics, and multi-user accounts.

Keep camera monitoring, evidence-backed review, and one reliable virtual removal. Those three pieces tell the complete product story.
