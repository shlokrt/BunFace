# FaceAuth Linux — Architecture

## System overview

```
┌──────────────────────────────────────────────────────────┐
│                    Linux PAM Stack                       │
│                                                          │
│  /etc/pam.d/sudo          /etc/pam.d/gdm-password        │
│       │                          │                       │
│  pam_exec.so               pam_exec.so                   │
│  auth.py                   face-login (bash)             │
│       │                          │                       │
│  auth_gui.py              gdm_auth_gui.py                │
│  (popup dialog)            (fullscreen overlay)          │
│       │                          │                       │
│       └──────────┬───────────────┘                       │
│                  │                                       │
│            liveness.py                                   │
│         (shared module)                                  │
└──────────────────────────────────────────────────────────┘
```

## Authentication pipeline

```
Frame from camera
       │
       ▼
MediaPipe BlazeFace
(face detection, 7ms/frame)
       │
       ├── No face → skip frame
       │
       ▼
Silent-Face MiniFASNet (×2 ensemble)
(liveness detection)
       │
       ├── Spoof → ignore frame (up to 4 ignored before blocking)
       │
       ▼
dlib face_recognition
(128-d embedding comparison)
       │
       ▼
Rolling window (10 frames)
       │
       ├── < 70% match → keep scanning
       │
       └── ≥ 70% match → ACCESS GRANTED
```

## File responsibilities

| File | Runs as | Called by | Purpose |
|---|---|---|---|
| `auth.py` | root | PAM | Detects display, launches GUI or headless |
| `auth_gui.py` | user | auth.py | PyQt5 popup for sudo |
| `gdm_auth_gui.py` | user | face-login | Fullscreen login overlay |
| `liveness.py` | user | auth_gui, gdm_auth_gui | Anti-spoofing (shared) |
| `register.py` | user | manual | One-time face registration |
| `face-login` | root→user | PAM | Sets env, runs gdm_auth_gui as user |

## Rolling window algorithm

```python
window = deque(maxlen=10)    # sliding window of bool

for each frame:
    if no face detected:    continue
    if spoof detected:      spoof_count += 1; continue
    
    dist    = euclidean(known_embedding, current_embedding)
    matched = dist < 0.6
    window.append(matched)
    
    if len(window) == 10:
        ratio = sum(window) / 10
        if ratio >= 0.7:
            grant_access()
        # else: window slides, keep scanning
```

## Anti-spoofing

Two MiniFASNet models run in parallel:
- `2.7_80x80_MiniFASNetV2` — wide receptive field, catches lighting artefacts
- `4_0_0_80x80_MiniFASNetV1SE` — deep features, catches pixel/ink patterns

Both output `[spoof_prob, real_prob, spoof_type2_prob]`. Index 1 (real_prob) is averaged. If average < 0.6 → spoof.

## PAM integration

```
auth optional pam_exec.so expose_authtok quiet /usr/local/bin/face-login
auth required pam_unix.so try_first_pass
```

- `optional` — face auth failure is never fatal
- `expose_authtok` — passes the password token (needed for pam_unix fallback)
- `quiet` — suppresses PAM noise on success
- `pam_unix.so` — standard password, always available as fallback
