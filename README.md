# VantaWatch AI

VantaWatch is a classroom CCTV safety demo that combines YOLO person tracking, weapon detection, pose and motion rules, an explainable LOW/MEDIUM/HIGH threat score, medical heuristics, evidence capture, and local police/hospital dashboards. Human review is required before any real response.

## Architecture

```text
Camera/video -> person, weapon, pose detectors -> behavior fusion -> threat engine
                                                    |                  |
                                                    +-> medical detector+
                                                       -> evidence manager
                                                       -> dispatcher -> notifier -> Flask server
                                                                          -> police / hospital dashboards
Phone GPS ---------------------------------------------------------------> server
```

## Folder layout

```text
main.py, config.py, *_detector.py, behavior_*.py, threat_engine.py
dispatcher.py, notifier.py, replay_outbox.py, evidence_manager.py, incident_logger.py
server.py, simulate_incident.py, selftest.py, selftest_pipeline.py, replay.py, make_demo.py
run_server.sh, run_demo.sh, run_dashboard_demo.sh, demo.bat, demo_dashboard_only.bat, play_demo.bat
weights/best.pt, yolov8n.pt, yolov8n-pose.pt
evidence/images/, evidence/videos/, evidence/incidents/
```

## Install and run

Use Python 3.12 in a virtual environment, then install the listed packages:

```powershell
python -m venv .venv
.venv\Scripts\activate
pip install -r requirements.txt
python server.py
```

The server uses port **5050** by default. Set `SV_PORT` to choose another port, for example `$env:SV_PORT=5051` in PowerShell or `SV_PORT=5051` before a shell command. `python server.py --port 5051` is also supported. Open `http://127.0.0.1:5050/` for Home. The top navigation links Home, Monitoring, Police Dashboard, Hospital Dashboard, Admin Console and GPS Sender. The Admin Console is `/admin` (also available as `/system`); Police and Hospital response workspaces are `/police` and `/hospital`; `/monitoring` is the video analysis workspace; `/alerts` shows the dedicated event queue.

On `/monitoring`, use the visible **Choose videos** file control to select one or more local MP4/M4V/MOV/AVI/MKV/WEBM files (up to 500 MB each); the selected filename/count appears beside it before upload. Fast mode uses the first 300 frames by default; change the cap or choose Full video. Jobs run in a single-worker queue with progress, cancel-current, skip-selected, clear-queued, annotated output and alert summaries. Use **Check cameras 0–3** to probe the camera backends supported by the current OS before starting the webcam attached to the Flask host. On macOS, grant camera access to the terminal or editor launching Flask (see below). Stop releases the capture. Reported FPS is processing throughput; it is not the clip's encoded frame rate. CPU analysis may run slower than real time.

For a command-line run, use `python main.py 0` or `python main.py evidence/videos/20260806_163322.mp4`. Press **q** or Esc to stop. Headless use: `python main.py evidence/videos/20260806_163322.mp4 --no-display --max-frames 70 --save-annotated verification/demo.mp4`. Browser audio unlocks on the first click or keypress anywhere on Home, Monitoring, Police or Hospital; after that, confirmed HIGH police alerts use the siren and medical alerts use a softer tone. A small speaker toggle mutes; Acknowledge records the status and stops the alert. Test controls are not shown in normal views. Developer-only simulations require `/?dev=1` and remain labelled SIMULATED. These are interface tests, never detections.

If alerts accumulated in `evidence/outbox.jsonl` while the server was down, restart the server and run `python replay_outbox.py`; the server deduplicates POSTs by incident id. The outbox tool accepts `--port` or `--server-url` for a non-default server address. `replay.py` is model-free local evaluation and does not contact the server.

For phone GPS, connect the phone and laptop to the same Wi-Fi, run `python server.py --https` (requires `pip install cryptography`), visit `https://<laptop-ip>:5050/gps` (or the configured port), accept the local certificate warning, and press Start. Keep the page open and allow location permission. A desktop without GPS hardware may provide no fix or only approximate network location.

## Running on macOS

The default port is 5050 because macOS may use port 5000 for AirPlay Receiver. You can change it with `SV_PORT`, and the server, simulator and outbox replay use the same configured port.

```bash
cd /path/to/smart-vision
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
pip install -r requirements.txt
chmod +x run_server.sh run_demo.sh run_dashboard_demo.sh
./run_server.sh
```

Open `http://127.0.0.1:5050/`, `/monitoring`, `/police`, `/hospital`, or `/gps` in a browser. Use `SV_PORT=5051 ./run_server.sh` to select a different port. The runner prefers CUDA when available, otherwise Apple MPS when PyTorch reports it available, then CPU. If an MPS model operation errors, that operation is retried on CPU and subsequent operations stay on CPU. The startup log prints the selected device. Intel Macs normally use CPU; MPS is for supported Apple Silicon builds.

For the webcam, start the app from Terminal or VS Code, attempt camera access, and allow that host app under **System Settings > Privacy & Security > Camera**. If macOS denied permission before, quit and reopen Terminal/VS Code after changing permission. The camera probe uses AVFoundation on macOS and reports this permission guidance when no camera opens. The app writes MP4 evidence using OpenCV's `mp4v` codec.

The local medical siren uses `afplay` with the built-in `Sosumi.aiff` sound; Windows uses `winsound`, and other systems use the terminal bell. Browser sirens still require one initial user interaction to unlock audio.

Expect CPU analysis to be slower than the source video. On the tested Windows CPU the fall clips ran about 0.56–1.34 FPS; earlier CPU runs were generally around 1–3 FPS. These are reference measurements, not a macOS performance promise. Apple Silicon MPS performance has not been measured on this machine; use the FPS shown by the app for your device. `./run_demo.sh` opens the offline clip and response dashboards; `./run_dashboard_demo.sh` starts the dashboard simulation. Simulated alerts are explicitly labeled.

## Limitations

- CPU inference is slow and is not real-time; webcam mode may lag.
- The third-party weapon detector has documented false positives and is not suitable for dispatch decisions.
- Blood detection is a color heuristic; pose/fall and running behaviour are not validated for real-world use.
- Dataset labels are clip-level only; object-level precision/recall is not established.
- Webcam access depends on the Flask host computer, Windows permissions and device availability. A remote browser cannot use its own camera through the server-side pipeline.
- GPS sharing requires location hardware/permission and a secure context for phones on the LAN.

## Final verification snapshot

`python selftest.py` passed 25 tests, including queue controls, camera-backend selection with a mocked device, siren unlock/alert hooks, and a mocked web-analysis fixture that verifies incident creation, server-siren invocation and the Police dashboard API. The pistol in that end-to-end fixture is synthetic test data, not a real detection. macOS-specific camera, MPS and `afplay` paths still require a Mac to verify.

A single real-model queue run processed `verification/cli_smoke.mp4` (2 decoded frames), `videos/weaponized/t_w001_converted.avi` (8 frames) and `videos/normal/t_n001_converted.avi` (8 frames). All finished with zero alerts. Processing throughput was about 0.16, 1.16 and 0.80 FPS; model construction took 0.53 seconds once. Time from request to first frame was 12.26, 14.33 and 21.25 seconds, including queue wait for later files. The earlier tiny-clip first-frame measurement was 88.64 seconds; this run measured 12.26 seconds for that same tiny fixture. No further YOLO run was made.

The real camera probe tried indexes 0–3 with DirectShow, Media Foundation and default backends; none opened. OpenCV reported “device did not open” and an index-out-of-range diagnostic, which does not establish whether the camera is missing, busy or blocked by permissions. The in-process Flask test client renders the updated Monitoring page, but the browser at `127.0.0.1:5050/monitoring` still showed the earlier page. Restart the server from this project folder and confirm the page shows **Choose videos** and **Threat engine**. Responsive screenshots at 1920×1080, 1366×768 and 390×844, live camera capture and audible browser playback remain unverified.

## Threat scoring

| Component | Score |
|---|---:|
| Weapon | For each: 40 + 40 × confidence; +20 if its box touches/overlaps a person box; maximum, capped at 100 |
| Behavior | Running 35; Raised Hand 30; Fall Detected 15; Aggressive Movement 65; Armed Person 60; Possible Attack 90 |
| Crowd panic | +15 to behavior score for at least 3 Running/Aggressive people; capped at 100 |
| Combined | 0.6 × higher sub-score + 0.4 × lower sub-score |
| Level | LOW below 30; MEDIUM 30–64; HIGH 65+ |

Scores are kept as two independently visible sub-scores so a reviewer can see whether a weapon or behavior drove the result. A weapon box that touches a person box receives the association bonus; an `Armed Person` or `Possible Attack` behavior can raise the combined score to HIGH. A weapon at confidence 0.5 with no nearby person is MEDIUM and cannot trigger police by itself.

## Tuning

Edit `config.py`. If running is missed, lower `RUNNING_SPEED`; if ordinary walking triggers, raise it. If falls are missed, adjust `FALL_ANGLE_DEGREES` carefully and test multiple camera angles. If red clothes trigger the blood heuristic, narrow its allowed pixel ratio/range or disable the heuristic for that camera. Also tune the 640-pixel input width and weapon confidence against the actual camera.

| Setting | Current value | Purpose |
|---|---:|---|
| `DEVICE` | `auto` | Uses CUDA when available, otherwise CPU |
| Person / weapon / pose image size | 640 / 640 / 640 px | Inference sizes; smaller isolated models were faster but did not improve full-pipeline speed reliably |
| `PERSON_TRACKER` | `botsort.yaml` | ByteTrack trial did not improve measured full-pipeline FPS |
| Person/pose cadence | every 3 frames | Reduces person/pose work; motion uses the source video timeline |
| Weapon cadence | every frame | Preserves short candidate events |
| Person / pose confidence | 0.30 / 0.40 | Detector thresholds |
| `WEAPON_CONFIDENCE` | 0.40 | Lowest threshold in the requested cached sweep; no Weaponized police alerts were produced in the reduced 5-clip check |
| `WEAPON_CLASS_CONFIDENCE` | pistol 0.40; knife 0.50 | Removes 3 of 6 low-threshold knife boxes in the reduced cache; higher-confidence scene-geometry boxes remain a known risk |
| `WEAPON_MAX_BOX_AREA_RATIO` | 0.10 | Reject weapon boxes over 10% of the frame area |
| `WEAPON_REQUIRE_PERSON_CONTACT_FOR_POLICE` | `True` | A weapon must touch/overlap a person box to permit a police alert |
| `WEAPON_CONFIRM_FRAMES` | 2 | Require two overlapping detections before weapon scoring |
| `HOLD_MARGIN` | 0.15 | Expand person boxes by 15% for weapon association |
| `WALKING_SPEED` / `RUNNING_SPEED` / `AGGRESSIVE_SPEED` | 0.8 / 1.4 / 3.0 body-heights/s | Motion classes |
| `FALL_ANGLE_DEGREES` / `KEYPOINT_CONFIDENCE` | 60° / 0.25 | Pose geometry and keypoint filter |
| `POLICE_CONFIRM_FRAMES` / `POLICE_COOLDOWN_SECONDS` | 6 / 30 s | Police alert confirmation and cooldown |
| `MEDICAL_FALL_SECONDS` / `MEDICAL_BLOOD_SECONDS` | 2.0 / 1.5 s | Medical persistence |
| Medical safety rules | Age ≥1 s; box height ≥15% frame; edge margin 1%; overlap <45%; stationary speed ≤0.35; pose fall needs ≥6 keypoints at confidence ≥0.50 | Blood color supports a confirmed down-person case and cannot alert alone |
| Evidence pre/post/max | 3 / 5 / 60 s | Clip buffer and recording limits |
| `EVIDENCE_MAX_GB` | 5 GB | Delete oldest unreferenced evidence only; incident-linked media is retained |

## How false alarms are reduced

The current filters use per-class confidence thresholds, a 10% maximum weapon box area, two-frame persistence, and weapon-person box contact before a police alert. The replay sweep retained the most Weaponized-folder detections at 0.40, but produced no Weaponized police alerts in the reduced 5-clip sample; increasing confidence erased nearly all candidates. The selected point is provisional and does not meet a verified Weaponized detection target. Medical alerts now require a stationary, sufficiently large, in-frame person with persistent fall evidence; red pixels alone never alert. See the dated results and filter effects in `VERIFICATION.md`. Keep a human in the loop.

## Responsible use

Alerts are decision support. A human operator must inspect the live scene and evidence before dispatch. Use only where surveillance is lawful, disclose monitoring, restrict access to recordings, and set retention to match the site's policy. The 5 GB cap removes only media not referenced by incident JSON; linked evidence is preserved, so unresolved incidents can exceed the cap.

## Five-minute examiner demo

1. Start the server and show both dashboards.
2. Run `python simulate_incident.py police`; show police incident, status actions, location link and image/video status.
3. Run `python simulate_incident.py medical`; show the hospital queue and medical reasons.
4. Start the GPS page on a phone (HTTPS) and show the live fix; demonstrate the fixed-camera fallback by stopping updates.
5. Run `play_demo.bat`. It opens the police and hospital pages with an offline banner and plays the quiet Normal clip labeled “RECORDED CLIP ANALYSED OFFLINE.” This sample has no alerts. No visually confirmed genuine weapon alert was found, so do not show a weapon-positive alert as genuine.

The earlier 30-clip model-backed passes at 0.40 and 0.75 are historical results. The latest reduced replay uses 5 clips/folder at 150 cached frames/clip; it produced 0/5 Weaponized police alerts, 0/5 Normal police alerts, and 1/5 Violence police alert. The one alert frame was visually uncertain, not a verified weapon.

## Limitations and future work

The SCVD sample exposed weak weapon localization and a poor threshold tradeoff. In the latest reduced cache, no threshold in the requested sweep produced a Weaponized police alert; one Violence alert remained and was visually uncertain. Current medical replay produced 0 alerts in 5 Normal clips, but the reduced sample does not revalidate the old 10-clip target. CPU throughput is slow and the full held-out run timed out. The sample has clip labels only, not frame-level ground truth, so it cannot establish object precision, weapon recall, blood accuracy, fall accuracy, or stable tracking IDs. The dashboard has no authentication and is demo-only. Human review is mandatory. See `VERIFICATION.md` for exact results and skipped work.

## Verification notes

`python selftest.py` currently passes 22 unit/API/integration checks plus the mocked frame-loop scenario. `VERIFICATION.md` records the previous 30-clip runs and the latest cached reduced-sample replay. The sample is class-labeled but has no frame-level ground truth.

The tested machine had Python 3.12.6, CPU-only PyTorch 2.12.1+cpu (`cuda.is_available=False`, 2 torch threads), an Intel Core i3-1005G1 CPU, and no GPU. The final 100-frame headless run measured 1.13 FPS. The pipeline is not real-time on this machine.
The project contribution is the surrounding tracking, pose/motion rules, threat and medical logic, filters, evidence workflow, replay evaluation, and dashboards. The pretrained detector weights are an external component.

## Current cached-round viva answers

The cached run used the first five sorted staged clips in each folder, up to 150 frames each. It stored 2,132 frames and averaged 2.38 FPS over 896.9 seconds of summed per-clip processing time (model startup excluded). The full clip list and filter comparison are in `VERIFICATION.md`.

| Confidence | Weaponized detections / police | Normal detections / police | Violence detections / police |
|---:|---|---|---|
| 0.40 | knife 6, pistol 35 / 0 | pistol 2 / 0 | pistol 148 / 1 |
| 0.50 | knife 3, pistol 11 / 0 | none / 0 | pistol 28 / 0 |
| 0.60 | none / 0 | none / 0 | pistol 2 / 0 |
| 0.70 | none / 0 | none / 0 | none / 0 |

At 0.40, the combined per-class thresholds, 10% box-area cap, contact gate and two-frame persistence changed the counts to Weaponized pistol 35/knife 3 with 0 police, Normal pistol 2 with 0 police, and Violence pistol 147 with 1 police. The alert frame was visually uncertain. The held-out real-model check stopped after `t_w011` timed out at 240 seconds; all remaining held-out clips were skipped.

- **Why the threshold trade-off?** The 0.40 point retained the most cached detections among the requested sweep, while 0.50–0.70 removed nearly all Weaponized candidates. Even at 0.40 there were no Weaponized police alerts in this reduced set, so this is a provisional operating point, not a successful detector result.
- **Why require a weapon near a person for police alerts?** Box contact prevents a weapon-like object on a rail or floor, with no nearby person, from triggering police. The object can still produce MEDIUM activity and evidence. The rule cannot correct a false weapon box that overlaps a pedestrian.
- **Why held-out testing?** It checks whether a chosen configuration carries to clips that were not used for tuning. The requested held-out run timed out on its first clip and was stopped; there are no held-out performance results.
- **What is the project’s contribution?** It is the tracking/pose/motion and alert logic, configurable safety filters, cached replay workflow, evidence capture, and dashboard integration. `best.pt` is third-party pretrained code/weights; the project did not train it.
- **Why decision support and not real-time?** The CPU cache pass averaged 2.38 FPS and one 150-frame main-pipeline attempt exceeded 240 seconds. Alerts need human review; speed and clip-level labels do not support reliable real-time dispatch.

## Demo confidence

Show the quiet Normal offline playback, the offline dashboard banner, the simulator workflows, and the medical unit-test scenarios. Avoid presenting the uncertain Violence alert or pedestrian/rail boxes as real weapon detections. No current alert frame was visually confirmed as a genuine weapon.

### Three weakest points

1. The reduced cache produced no Weaponized police alerts; its one alert was an uncertain Violence object.
2. The held-out run timed out on its first clip, so generalization is unknown.
3. CPU speed is low and validation has clip labels only; fall, running and real weapon behavior remain unverified.
# Behavior and weapon output notes (2026-11-20)

Behavior labels are motion/pose heuristics, not outputs from a trained violence classifier. A person is shown as **Unknown** until the tracker provides enough motion samples; later labels include the measured speed in body-heights per second. This avoids calling a first sighting or a tracking gap “Standing.” The revised motion calculation includes filtered path movement as well as net displacement, which is intended to better capture motion that changes direction. It has synthetic self-test coverage but has not yet been confirmed on a post-change real-video run.

Weapon model boxes are candidates for human review, not proof of a weapon. The provided third-party weights have produced false boxes on people and scene structures. No model file or confidence/persistence filter was changed for the behavior update. On the one reviewed 60-frame Violence diagnostic, the model produced a pistol candidate and a police alert, but the examined frames did not clearly show a weapon. Treat that result as unconfirmed. See `VERIFICATION.md` for the run limits and evidence.

## Fall clips and medical siren

The three user supplied fall clips were each processed once on CPU and are saved under `verification/fall_samples/` with annotated outputs. They produced zero medical incidents, so no siren sounded. The siren is dispatched when the medical detector emits a confirmed MEDICAL incident; the code path is covered by self-tests. The supplied clips show that fall recognition itself remains unreliable: pose labels missed the down-person posture in reviewed frames, and one reviewed person was clipped by the frame edge, which the current medical safety filter excludes. See the fall-video table in `VERIFICATION.md`. Do not claim these clips demonstrate successful fall detection or siren triggering.

