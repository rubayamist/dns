# CFML — Count my fingers using ML

A web page that watches your webcam and reports how many fingers you are
holding up. HTML, CSS and JavaScript only; Python is used just to download the
model once and to serve the folder.

## Put it in place

The zip holds a single `mywebsite` folder, so extract it to `C:\` and you get:

```
C:\mywebsite\
  index.html
  run.bat
  serve.py
  setup.py
  make_cert.py
  css\style.css
  js\app.js
  js\fingers.js
  models\          <- hand_landmarker.task lands here
  vendor\          <- MediaPipe runtime lands here
  certs\           <- only if you choose the HTTPS option
```

## The three addresses

The server listens on every network interface at once, so once it's running,
all three of these reach the same page — pick whichever you like:

| Address | Needs the hosts file? | Camera works right away? |
| --- | --- | --- |
| `http://127.0.0.1` | no | **yes** |
| `http://192.168.0.111` | no, it's your machine's own LAN address | no — see below |
| `http://rubaya.com` | yes, one line in hosts | no — see below |

`127.0.0.1` and `localhost` are the only addresses a browser treats as secure
by default. Everything else — a custom hostname like `rubaya.com`, or a plain
LAN IP like `192.168.0.111` — is just an ordinary HTTP address as far as the
browser's camera permission is concerned, secure-feeling as it may look on
your own network. The fixes below apply to both of them together.

## Point rubaya.com at your machine

Skip this section entirely if you only plan to use `127.0.0.1` or the LAN IP.

Open Notepad **as administrator**, then open

```
C:\Windows\System32\drivers\etc\hosts
```

Add this line at the bottom and save:

```
127.0.0.1 rubaya.com
```

Check it took effect:

```
ping rubaya.com
```

It should answer from `127.0.0.1`.

## Run it

Right-click **run.bat** and choose **Run as administrator** — port 80 normally
needs it. On the first run it downloads the model (about 12 MB), then serves the
folder on port 80. The console lists all three addresses; it opens
`http://127.0.0.1` automatically since that one needs no further setup.

By hand, from a Command Prompt in `C:\mywebsite`:

```
python setup.py       (once, needs internet)
python serve.py       (port 80, as administrator)
```

Do not open `index.html` by double-clicking it. On a `file://` address browsers
refuse both the camera and JavaScript modules.

## Making the camera work at rubaya.com or the LAN address

If you only use `http://127.0.0.1`, nothing here applies — it already works.
For `rubaya.com` and `192.168.0.111`, pick one of these.

**Option A — tell Chrome to trust those addresses.** Close every Chrome window
first, then run this as a single line (both addresses, comma-separated, no
spaces around the comma):

```
chrome.exe --unsafely-treat-insecure-origin-as-secure="http://rubaya.com,http://192.168.0.111" --user-data-dir="%TEMP%\cfml-profile"
```

The separate profile keeps the exception away from your normal browsing.
`serve.py` prints this exact command, filled in with whatever address it
detected, every time it starts. Nothing to install, but you need the command
every time.

**Option B — serve it over HTTPS.** One-time setup:

```
pip install cryptography
python make_cert.py
certutil -addstore -f "ROOT" certs\cert.pem     (as administrator)
```

`make_cert.py` detects this machine's LAN address on its own and bakes it into
the certificate alongside `rubaya.com`, `localhost` and `127.0.0.1`. If the
address ever changes, or you want to be sure, pass it explicitly:

```
python make_cert.py 192.168.0.111
```

After that, `run.bat` finds the certificate on its own and serves everything
over HTTPS on port 443 — `https://rubaya.com`, `https://192.168.0.111`,
`https://127.0.0.1` — with the camera working in any browser and no flags.
Remove the trust later with `certutil -delstore "ROOT" rubaya.com`.

## If port 80 is taken

Windows often gives port 80 to IIS or the World Wide Web Publishing Service. See
what holds it:

```
netstat -ano | findstr :80
```

Free it, as administrator:

```
net stop http
net stop w3svc
```

Or sidestep it entirely — the hosts entry still applies:

```
python serve.py 8080          then open http://rubaya.com:8080
```

## How it works

1. **MediaPipe HandLandmarker** finds up to two hands per frame and returns 21
   3D landmarks for each — wrist, knuckles and the joints of every finger. The
   model file is `models/hand_landmarker.task`.
2. **`js/fingers.js`** turns those points into a count. For the four fingers it
   measures the angle at the middle joint: straighter than 150° counts as up.
   The thumb bends sideways instead of curling, so it gets its own test —
   straight at its own joint, and tip further from the far edge of the palm
   than its base.
3. A rolling vote over the last six frames stops the number flickering while a
   finger is mid-move.

Angles and ratios are used rather than pixel positions, so tilting the hand or
moving nearer the camera does not change the answer.

## Tuning

Open `js/fingers.js` and edit `TUNING`:

| Setting | Effect |
| --- | --- |
| `fingerAngle` | Higher means a finger must be straighter to count |
| `thumbAngle` | Same, for the thumb |
| `thumbSpread` | Higher means the thumb must stick out further |
| `voteFrames` | More frames: steadier number, slower to react |

## If something goes wrong

**"Open through the local server"** — the page was opened as a file. Use
`run.bat` and the rubaya.com address.

**"Origin is not secure"** — expected on `http://rubaya.com`. Use option A or B
above.

**rubaya.com does not resolve** — the hosts line was not saved, usually because
Notepad was not run as administrator. Check with `ping rubaya.com`, and flush the
cache with `ipconfig /flushdns`.

**"Camera blocked"** — another app is holding the camera, or the site
permission is denied. In Chrome, click the icon at the left of the address bar
and allow the camera, then reload.

**"Model failed to load"** — `python setup.py` has not run yet and there is no
internet connection to fall back on.

**Counts one finger too many or too few** — light the hand from the front, keep
fingers apart, and avoid pointing a finger straight at the lens; a foreshortened
finger looks folded to any landmark model. Then adjust `TUNING`.

**Slow** — close other camera or GPU-heavy tabs. The frame rate is shown on the
right of the page.

## Model and licence

The model is Google's MediaPipe HandLandmarker, trained on roughly 30,000 real
hand photographs plus rendered synthetic hands, published under Apache 2.0. The
counting logic, page and styles here are yours to change freely.

Video never leaves your machine; every frame is processed in the browser.
