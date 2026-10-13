# Firebase setup for online play

The game uses Firebase Realtime Database for room state and Anonymous Authentication for
player identity. The host's browser runs the game logic; guests send moves through the
database. No application server code is needed.

All rules and configuration examples are included below. The console setup needs no local
Firebase configuration or rules files. Firebase Hosting is optional: the game can run on
GitHub Pages or another static HTTP/HTTPS host.

## 1. Create or select the project

Open the [Firebase console](https://console.firebase.google.com/) and select the project
you want to use, or choose **Add project**. Google Analytics is not needed for this game.
Use the same project for the web app, Authentication, and Realtime Database.

## 2. Register a web app

Project overview -> `</>` (Web) -> register the app. If the app is already registered, go to
Project settings -> General -> Your apps -> the web app -> SDK setup and configuration ->
Config. Copy its `firebaseConfig` values for step 7.

The game already imports the Firebase JavaScript SDK from Google's CDN when online play is
used. There is no npm dependency or extra Firebase script tag to add to `index.html`.

## 3. Turn on Anonymous sign-in

Authentication -> Get started (if shown) -> Sign-in method -> Anonymous -> Enable -> Save.
Players do not need to enter an email address or password. The game calls
`signInAnonymously` before accessing the database.
[Firebase anonymous authentication documentation](https://firebase.google.com/docs/auth/web/anonymous-auth).

## 4. Create the Realtime Database

Realtime Database -> Create database. Choose **Realtime Database**, not Cloud Firestore.
Pick a region near the players and start in **locked mode**. Step 5 replaces the locked
rules with the game's rules.

Copy the exact database URL shown on the Data tab. Depending on the database region, it
can look like either of these:

- `https://DATABASE_NAME.firebaseio.com`
- `https://DATABASE_NAME.REGION.firebasedatabase.app`

Do not infer the URL from the project ID or use a Firebase Hosting URL. Include this value
as `databaseURL` in step 7.
[Firebase database setup documentation](https://firebase.google.com/docs/database/web/start).

## 5. Publish the rules

Realtime Database -> Rules -> replace the editor contents with the complete JSON below ->
**Publish**. Select the database instance whose URL you copied in step 4.

```json
{
  "rules": {
    ".read": false,
    ".write": false,
    "rooms": {
      "$code": {
        ".read": "auth != null",
        ".write": "auth != null && (!data.exists() || data.child('host').val() === auth.uid)",
        "seats": {
          "$seat": {
            ".write": "auth != null && (data.child('uid').val() === auth.uid || (!data.child('uid').exists() && newData.child('uid').val() === auth.uid))"
          }
        },
        "intents": {
          "$id": {
            ".write": "auth != null && !data.exists() && newData.child('uid').val() === auth.uid"
          }
        },
        "chat": {
          "$id": {
            ".write": "auth != null && !data.exists() && newData.child('uid').val() === auth.uid",
            ".validate": "newData.child('text').isString() && newData.child('text').val().length <= 50"
          }
        }
      }
    },
    "lobbies": {
      ".read": "auth != null",
      "$code": {
        ".write": "auth != null && (!data.exists() || data.child('host').val() === auth.uid)"
      }
    }
  }
}
```

These rules match the current game:

- Signed-in users can read an individual room and the public lobby list.
- Signed-in users can create rooms; the recorded host can update an existing room and
  its state, and remove processed guest intents.
- Guests can claim an empty seat, update their own seat, and submit new intents and chat
  messages carrying their own UID. Chat text is limited to 50 characters.
- The recorded host can update or remove an existing public lobby listing. Keep the
  `/lobbies` rules even if joining by code already works.

Room reads, intents, and chat are not restricted to room members by these rules. A room
code is a way to find a room, not an access-control boundary. The host is trusted to run
the game correctly. Parent read/write grants also apply to child paths; child write rules
do not remove the host's room-level access.
[Firebase rules semantics](https://firebase.google.com/docs/database/security/core-syntax).

### Optional: publish the same rules with the CLI

Skip this subsection when publishing through the console. For CLI deployment, use a
current Node.js LTS installation with npm, then save the complete rules JSON above as
`database.rules.json` and the following configuration as `firebase.json` in the same folder:

```json
{
  "database": {
    "rules": "database.rules.json"
  }
}
```

Run these commands from that folder, replacing `YOUR_PROJECT_ID` with the actual project ID:

```powershell
npx firebase-tools login
npx firebase-tools deploy --only database --project YOUR_PROJECT_ID
```

This deploys Realtime Database rules only; it does not publish `index.html` or enable
Anonymous Authentication. CLI deployment overwrites the rules published in the console,
so keep the inline rules and any local rules copy in sync before deploying.
[Firebase CLI documentation](https://firebase.google.com/docs/cli).

## 6. Authorise the site if required

If the game reports `auth/unauthorized-domain` or an "authorised list" error, open
Authentication -> Settings -> Authorized domains -> Add domain. Use the hostname only,
without a protocol, port, or path:

- `rezformbay.github.io` for `https://rezformbay.github.io/Catan/`
- `mehrbat.github.io` for `https://mehrbat.github.io/Catan/`
- Your custom site's hostname, if used
- `localhost` for local development, if needed

Anonymous sign-in does not use an OAuth redirect, so this is normally an error-specific
step. Do not assume `localhost` is already listed: projects created after April 28, 2025
no longer include it by default.
[Firebase authorised-domain documentation](https://firebase.google.com/docs/auth/web/email-link-auth).

## 7. Paste the config

Replace the existing `FIREBASE_CONFIG` object in `index.html` (section 10, Online play) with
the values from step 2 and the exact `databaseURL` from step 4. Replace every placeholder:

```javascript
const FIREBASE_CONFIG = {
  apiKey: "YOUR_FIREBASE_WEB_API_KEY",
  authDomain: "YOUR_PROJECT_ID.firebaseapp.com",
  databaseURL: "YOUR_EXACT_REALTIME_DATABASE_URL",
  projectId: "YOUR_PROJECT_ID",
  storageBucket: "YOUR_STORAGE_BUCKET_FROM_THE_CONFIG",
  messagingSenderId: "YOUR_MESSAGING_SENDER_ID",
  appId: "YOUR_WEB_APP_ID"
};
```

Copy `authDomain` and the other values from Firebase rather than guessing them.
`measurementId` is optional, and the game does not use Analytics or Cloud Storage.
The online controls check that `apiKey` and `databaseURL` are nonempty; placeholder values
will pass that check but fail when connecting.

The Firebase web API key identifies the project and is designed to be present in browser
code. Database access is controlled by Authentication and Security Rules. Use the Firebase
web config, never a service-account private key or Admin SDK credential.
[Firebase API key documentation](https://firebase.google.com/docs/projects/api-keys).

Save and publish the updated `index.html` through your static site's normal deployment.
Reload the deployed page so it uses the new config. Serve local copies over HTTP rather
than opening the HTML as a `file://` URL.

## 8. Verify online play

Use two separate browser profiles, a normal window and a private window, or two devices.
Tabs in the same browser profile can share the same anonymous identity.

1. Open the updated game and host a public room in the first browser. Keep the host page
   open and connected for the whole game.
2. In Firebase Authentication -> Users, confirm that an anonymous user was created.
3. In Realtime Database -> Data, confirm `/rooms/ROOM_CODE` exists and the public room has
   a matching `/lobbies/ROOM_CODE` entry.
4. Join by room code from the second browser. Confirm both players appear. Also confirm
   the public room appears in the public list while it is open and has an empty seat.
5. Start the game, make a valid guest move, and send a chat message. Confirm both browsers
   receive the same state and the message appears in both.

The host broadcasts state every four seconds. There is no automatic host migration if
the host closes or reloads the page. Room records and chat are not automatically expired
by the current code; a public listing is scheduled for removal when its host disconnects.

## Troubleshooting

| Symptom | Check |
| --- | --- |
| Online controls stay disabled | Step 7: `apiKey` and `databaseURL` must both be filled in; reload the updated page. |
| Permission denied / `PERMISSION_DENIED` | Step 5: publish the complete rules to the database used by `databaseURL`, including `/lobbies`. Confirm Authentication and the database belong to the same project. |
| Anonymous sign-in switched off / `auth/operation-not-allowed` | Step 3: enable the Anonymous provider and save. |
| Site not on the authorised list / `auth/unauthorized-domain` | Step 6: add the exact hostname serving the game. |
| Invalid API key | Step 7: recopy the web app config from the intended project. If the key has HTTP referrer restrictions, check that the deployed site's URL is allowed. |
| Database URL error | Step 4: use the exact Realtime Database URL, including its region-specific hostname. |
| Could not reach Firebase / failed to fetch | Check connectivity and browser developer tools. Online play needs access to `www.gstatic.com` for SDK imports and to Firebase Auth and Database endpoints. |
| Room works by code but is not listed publicly | Host a public room and check the `/lobbies` rules and data in step 5. Started or full rooms are hidden from the public list. |
| Second player acts as the same identity | Step 8: use a separate browser profile, private window, or device. |
| Host has gone quiet / state stops syncing | Keep the host page connected. Host closure, suspension, or network loss stops authoritative game updates. |
| Too many sign-in attempts / `auth/too-many-requests` | Firebase limits new anonymous sign-ups from one IP in a short period. Stop repeated sign-in attempts and retry later. |

The messages under HOST / JOIN identify likely causes. For other errors, inspect the browser
console for the full Firebase error code and the failing operation.
