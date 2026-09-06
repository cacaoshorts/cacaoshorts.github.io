# Cacao Shorts — Google Play Data Safety audit

**Audit date:** September 6, 2026

**Scope:** Current Android client, committed Supabase schema/production patches and Edge Functions, and the public legal site.

**Purpose:** Internal Play Console entry worksheet. This document is not the Privacy Policy.

## Executive determination

- The app transmits user data off-device, so answer **Yes** to “Does your app collect or share any of the required user data types?”
- Declare these collected types: **Name, Email address, User IDs, App interactions, Other actions, and Device or other IDs**.
- Declare **Shared: No** for each collected type under the current architecture. Supabase and Firebase act as service providers, while Google sign-in is a user-initiated authentication action. This conclusion must be revisited if providers use the data for independent purposes or if advertising/analytics SDKs are added.
- All current app-controlled transmissions use encrypted HTTPS/TLS connections: answer **Yes** to encryption in transit.
- The app supports account creation. A permanent user can delete an account in **Profile → Delete Account**, and the external resource is <https://cacaoshorts.github.io/delete-account/>.
- Search text is filtered on-device after the published catalog is loaded. It is neither sent as a search parameter nor stored by Cacao Shorts, so **In-app search history: No**.
- Firebase Cloud Messaging auto-initialization is not disabled. Firebase therefore generates/transmits an app-installation identifier and registration information at startup, including for Guests before notification opt-in. Declare **Device or other IDs: Yes**.
- The current client has no analytics, Crashlytics, advertising, camera/photo upload, microphone, contacts, location, payments, or user-file SDK/path.

## Evidence summary

| Area | Actual behavior and evidence |
|---|---|
| Anonymous authentication | `AuthRepository.createAnonymousSession()` calls `signInAnonymously()`. Bootstrap calls it only when Guest Mode is enabled and no session exists. A Supabase Auth user ID and session are created. |
| Profiles | `handle_new_user()` creates `profiles`; Guests start with `is_anonymous=true`, null username and null avatar. Permanent onboarding stores a chosen username and an optional predefined `avatar_key`. No user photo upload exists. |
| Viewing activity | Player writes position, completion, playback timestamps, and watch count through the existing playback-progress RPC. `episode_progress` and `series_watch_state` back resume and Continue Watching. |
| My List | Series Detail inserts/deletes the current user’s `favorites` rows; My List reads those rows. |
| Ratings | `SeriesRatingRepository` reads and upserts a private 1–5 `series_ratings` row for the current authenticated user. |
| Search | `SearchPage` loads the published discovery catalog through `DiscoveryRepository`, then `_searchController` text is used only by local widget filtering. No search text is passed to Supabase or an analytics service. |
| Events | No Flutter code writes `public.events`. The similarly named `series_notification_events` table is a server-side notification delivery queue, not user analytics. |
| Notifications | `PushNotificationService.initialize()` initializes Firebase/FCM at app startup. The manifest does not disable FCM auto-init. Guests are blocked by `_hasPermanentUser`; only an opted-in permanent user sends an FCM token/platform/enabled/last-seen state to `push_devices`. |
| Reminder state | `SharedPreferencesGuestRegistrationReminderStore` stores only a SHA-256 Guest fingerprint and dismissal time on-device. It introduces no off-device collection. |
| Same-UID upgrade recovery | SharedPreferences stores conversion kind, step, and the internal source user ID so an interrupted email/OAuth conversion can resume. No password or JWT is stored there. |
| Existing-account merge recovery | `flutter_secure_storage` stores the opaque transfer credential, flow type, creation time, and step. The Edge request contains only that credential. No password, JWT, source UID, or destination UID is stored in this record or sent in its body. |
| Account deletion | Permanent Profile exposes a two-step permanent delete flow. The authenticated `delete-account` Edge Function derives the caller from the session and deletes that Auth user; cascading foreign keys remove profile, roles, progress, watch state, favorites, ratings, and push-device rows. |
| Google authentication | The public Google path uses `openid email` scopes. The app receives the account email/provider identity required for authentication. The current code does not request `profile`, does not copy provider names, and does not copy provider photos. |
| Facebook | Repository support exists with the `email` scope, but no public Facebook button/path is rendered. It is code-present but inaccessible in the current public UI and must not be represented as an available feature. |
| SDK automatic collection | Firebase documentation states that FCM auto-init generates registration/installation identifiers and uploads identifier/configuration data. No Firebase Analytics or Crashlytics package is present. |

## Data Safety type matrix

“Required” below follows Play’s rule that a type is required when users cannot turn off its collection. “Optional” means it is supplied only after a user chooses the corresponding action.

| Google Play category / type | Collected | Shared | Ephemeral | Required / optional | Purpose(s) | Evidence / impact |
|---|---:|---:|---:|---|---|---|
| Location — Approximate location | No | No | No | N/A | N/A | No location permission or location SDK; Cacao Shorts does not derive location from IP addresses. Ordinary service-provider network logs are disclosed in the Privacy Policy. |
| Location — Precise location | No | No | No | N/A | N/A | No permission, API, or code path. |
| Personal info — Name | **Yes** | No | No | Optional | App functionality; Account management | The user-chosen username is a nickname under Play’s “Name” definition. Guests do not choose one until conversion/onboarding. Google `profile` scope is not requested. |
| Personal info — Email address | **Yes** | No | No | Optional | Account management | Email sign-up/sign-in/recovery and Google `email` scope. Guests can use the app without email. |
| Personal info — User IDs | **Yes** | No | No | Required | App functionality; Account management; Fraud prevention, security and compliance | Supabase assigns every Guest/permanent user an internal ID used by RLS and ownership. Guest impact: temporary pseudonymous ID; permanent impact: account ID/provider identity linkage. |
| Personal info — Address | No | No | No | N/A | N/A | No field or code path. |
| Personal info — Phone number | No | No | No | N/A | N/A | No phone-auth or profile field. |
| Personal info — Race and ethnicity | No | No | No | N/A | N/A | No field or inference path. |
| Personal info — Political or religious beliefs | No | No | No | N/A | N/A | No field or inference path. |
| Personal info — Sexual orientation | No | No | No | N/A | N/A | No field or inference path. |
| Personal info — Other info | No | No | No | N/A | N/A | No other personal-profile field is requested. Predefined avatar choice is classified under Other actions. |
| Financial info — User payment info | No | No | No | N/A | N/A | No billing or payment SDK. |
| Financial info — Purchase history | No | No | No | N/A | N/A | No purchases. |
| Financial info — Credit score | No | No | No | N/A | N/A | No code path. |
| Financial info — Other financial info | No | No | No | N/A | N/A | No code path. |
| Health and fitness — Health info | No | No | No | N/A | N/A | No code path. |
| Health and fitness — Fitness info | No | No | No | N/A | N/A | No code path. |
| Messages — Emails | No | No | No | N/A | N/A | Account email addresses and transactional auth emails are not collection of users’ email-message content. |
| Messages — SMS or MMS | No | No | No | N/A | N/A | No SMS access. |
| Messages — Other in-app messages | No | No | No | N/A | N/A | No chat or messaging. |
| Photos and videos — Photos | No | No | No | N/A | N/A | No user photo upload or photo permission; avatars are bundled assets. |
| Photos and videos — Videos | No | No | No | N/A | N/A | Users stream catalog videos but do not provide/upload videos. |
| Audio files — Voice or sound recordings | No | No | No | N/A | N/A | No microphone permission or recording. |
| Audio files — Music files | No | No | No | N/A | N/A | No user audio access. |
| Audio files — Other audio files | No | No | No | N/A | N/A | No user audio access. |
| Files and docs — Files and docs | No | No | No | N/A | N/A | No file picker or document upload. |
| Calendar — Calendar events | No | No | No | N/A | N/A | No calendar permission/API. |
| Contacts — Contacts | No | No | No | N/A | N/A | No contacts permission/API. |
| App activity — App interactions | **Yes** | No | No | Required | App functionality | Episode IDs, playback position, completion, timestamps, and Continue Watching state are persisted for Guests and permanent users when they watch. |
| App activity — In-app search history | **No** | No | No | N/A | N/A | Search string stays in the `TextEditingController`; only catalog reads leave the device. No retained or ephemeral search-query transmission. |
| App activity — Installed apps | No | No | No | N/A | N/A | No package inventory access. The narrow Android intent query does not enumerate installed apps or transmit results. |
| App activity — Other user-generated content | **No** | No | No | N/A | N/A | No open-ended user posts, bios, comments, or reviews. A structured 1–5 rating is classified as Other actions. |
| App activity — Other actions | **Yes** | No | No | Optional | App functionality; Account management | My List changes, private 1–5 ratings, Continue Watching dismissal, predefined avatar choice, and opted-in notification preference/device registration. Guests use favorites/ratings; notification enrollment is blocked. |
| Web browsing — Web browsing history | No | No | No | N/A | N/A | OAuth opens the user’s external browser; Cacao Shorts does not collect browsing history. |
| App info and performance — Crash logs | No | No | No | N/A | N/A | No Crashlytics or other remote crash SDK. Local `debugPrint` is not transmitted by the app. |
| App info and performance — Diagnostics | No | No | No | N/A | N/A | No performance-monitoring/diagnostics SDK. |
| App info and performance — Other performance data | No | No | No | N/A | N/A | No code path. |
| Device or other IDs — Device or other IDs | **Yes** | No | No | Required | App functionality | FCM auto-init creates/transmits Firebase installation/registration identifiers at startup. For an opted-in permanent user, the FCM token is additionally associated with the account in `push_devices`. Guest server enrollment is blocked, but Firebase-side startup processing still occurs. |

## Sharing determination

The recommended Play answer is **no data shared**:

- Supabase hosts/authenticates/processes data on behalf of Cacao Shorts and fits Play’s service-provider exception.
- Firebase/Google processes installation IDs and notification registration data to provide the configured FCM service. This assessment treats that processing as service-provider processing, not independent advertising/analytics use.
- Google OAuth is explicitly user initiated and expected for authentication; it also falls within Play’s user-initiated-action exception.
- No advertising, cross-app tracking, sale, data-broker transfer, or analytics transfer was found.

Reassess this answer if contractual/provider use changes, Analytics/Crashlytics/ads are added, or Facebook is exposed publicly.

## Exact Play Console worksheet

### Data collection and security

| Play Console question | Recommended answer |
|---|---|
| Does your app collect or share any of the required user data types? | **Yes** |
| Is all user data collected by your app encrypted in transit? | **Yes** |
| Do you provide a way for users to request that their data is deleted? | **Yes** |
| Does the app allow users to create an account? | **Yes** |
| In-app deletion path | **Profile → Delete Account → confirmations → Delete permanently** (permanent accounts) |
| External deletion URL | **https://cacaoshorts.github.io/delete-account/** |

### Select these collected types

| Type | Collected | Shared | Required/optional | Purposes to select |
|---|---:|---:|---|---|
| Name | Yes | No | Optional | App functionality; Account management |
| Email address | Yes | No | Optional | Account management |
| User IDs | Yes | No | Required | App functionality; Account management; Fraud prevention, security and compliance |
| App interactions | Yes | No | Required | App functionality |
| Other actions | Yes | No | Optional | App functionality; Account management |
| Device or other IDs | Yes | No | Required | App functionality |

Do **not** select Analytics, Advertising or marketing, or Personalization as a purpose. Continue Watching and My List are direct app functionality, not behavioral advertising or generalized personalization in this implementation.

## Retention and deletion findings

- There is **no deployed fixed-schedule automatic cleanup** for abandoned anonymous Auth users or their Guest data. Supabase also documents that anonymous-user cleanup is not automatic.
- Public wording must therefore not promise deletion after 30, 60, or 90 days. Guest identity/data may remain until conversion/merge, an authenticated deletion path, manual support handling where the Guest can be safely identified, or future operational cleanup.
- Losing local app storage/session does not itself delete server Guest data and can prevent the person from reconnecting to it.
- Permanent account deletion is implemented in-app and removes the Auth user plus cascade-linked product rows.
- The current Guest Profile has no direct Delete Guest Data action. Once the Guest session is lost, an email request may be insufficient to locate the pseudonymous Guest safely. This is a **LAUNCH FOLLOW-UP REQUIRED** item.
- `public.events` is not written by the current Flutter client. It has no Auth foreign key, so any legacy user-linked event rows would not be removed by the current permanent-account delete Edge Function. Confirm production has no such rows or add tightly scoped cleanup before claiming universal deletion of all historical event data. This is a **LAUNCH FOLLOW-UP REQUIRED** item.

## Remaining release/compliance gates

1. **Guest → new Google same-UID conversion:** physical gate pending.
2. **Guest → existing Google account merge:** physical gate pending.
3. **Guest deletion control:** add a discoverable, authenticated Guest-session deletion path or another verifiable equivalent.
4. **Guest lifecycle:** adopt and deploy an explicit abandoned-Guest retention/cleanup policy; none exists today.
5. **Legacy events deletion:** verify there are no linked legacy rows or update permanent-account deletion to remove/anonymize them lawfully.
6. Enter and submit the Data Safety form manually; this audit does not alter Play Console.

## Authoritative references

- Google Play Data Safety definitions and service-provider/user-initiated exceptions: <https://support.google.com/googleplay/android-developer/answer/10787469>
- Google Play account-deletion requirement: <https://support.google.com/googleplay/android-developer/answer/13327111>
- Firebase privacy/service data, including FCM Firebase installation IDs: <https://firebase.google.com/support/privacy/>
- Firebase Cloud Messaging startup registration and auto-init: <https://firebase.google.com/docs/cloud-messaging/android/get-started>
- Supabase anonymous sign-ins and cleanup limitations: <https://supabase.com/docs/guides/auth/auth-anonymous>
