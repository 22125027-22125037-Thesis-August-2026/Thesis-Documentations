# Mobile App — `thesis-mobile`

| | |
|---|---|
| **Repo** | [`thesis-mobile`](https://github.com/22125027-22125037-Thesis-August-2026/thesis-mobile) (GitHub; local working copy: `D:\Y4-Sem 2 Thesis\thesis-mobile`) |
| **Platform** | React Native **0.83.1**, React **19**, TypeScript |
| **Audience** | Teens / patients |
| **Backend** | release: `BASE_URL = https://umatter-apcs.duckdns.org` (Caddy → gateway); dev: `http://<PUBLIC_IP>:8080` |

---

## 1. Purpose

The mobile app is the **teen/patient client** — the primary face of uMatter. It covers the full
patient journey: onboarding, self-care tracking, the AI companion, finding & booking a therapist,
attending video sessions, peer social chat, and notifications.

---

## 2. Tech stack

| Concern | Library |
|---|---|
| Navigation | `@react-navigation/native`, `native-stack`, `bottom-tabs` |
| HTTP | `axios` |
| Real-time chat | `@stomp/stompjs` + `text-encoding` (STOMP over WebSocket to Social `:8086`) |
| Push notifications | `@react-native-firebase/messaging` + `@notifee/react-native` |
| Video consultations | `react-native-webview` / `react-native-video` (Zoom join) |
| Charts | `react-native-chart-kit` (mood/sleep trends) |
| Calendars | `react-native-calendars` (booking slots) |
| Media | `react-native-image-picker` (diary/treasure/avatar uploads) |
| i18n | `i18next` + `react-i18next` (locales in `src/locales`) |
| JWT/crypto | `react-native-pure-jwt`, `crypto-js` |
| Storage | `@react-native-async-storage/async-storage` (token persistence) |
| Icons / SVG | `react-native-vector-icons`, `react-native-svg` |

Requires **Node ≥ 20**. Build commands: `npm run android` / `npm run ios`,
`npm run build:android` (release APK), `build:android:arm64`.

---

## 3. Source structure

```
thesis-mobile/
├── App.tsx                 # root: providers, navigation container
├── src/
│   ├── api/                # axios clients per backend domain (+ __tests__)
│   ├── navigation/         # stack + tab navigators
│   ├── screens/            # see screen map below
│   ├── components/         # home/, tour/, tracking/ + shared UI
│   ├── context/            # auth, i18n, app-wide state
│   ├── hooks/              # reusable hooks
│   ├── services/           # FCM, STOMP, storage, etc.
│   ├── constants/  theme/  types/  utils/
│   ├── locales/            # i18n translation files
│   ├── native/             # native module bridges
│   └── assets/             # audio, booking art, fonts, logo
├── android/  ios/          # native projects
├── patches/                # patch-package patches
└── mock/                   # mock data (e.g. notifications.json)
```

---

## 4. Screen map (feature → screens)

| Feature area | Screens |
|---|---|
| **Onboarding** | `SplashScreen`, `OnboardingScreen` |
| **Auth** | `LoginScreen`, `RegisterScreen`, `TermsScreen` |
| **Home** | `HomeScreen` (+ `components/home`, `components/tour`) |
| **Tracking** | screens in `screens/tracking` (+ `components/tracking`) — mood, sleep, food, diary, breathing, streaks, treasures |
| **AI companion** | `chat/ChatScreen`, `chat/ChatRoomScreen`, `chat/TherapyOverviewScreen` |
| **Therapy / booking** | `booking/MatchingFormScreen`, `TherapistDetailScreen`, `BookingScreen`, `AppointmentsHistoryScreen`, `ConsultationDetailScreen`, `ConsultationFeedbackScreen`, `WaitingRoomScreen`, `VideoConsultationScreen` |
| **Social** | `social/MessageListScreen`, `social/ChatScreen`, `social/FriendProfileScreen` |
| **Notifications** | `notification/NotificationScreen`, `NotificationDetailScreen` |
| **Profile** | `profile/ProfileScreen`, `ProfileEditScreen`, `AboutScreen`, `ContactScreen`, `FAQScreen` |
| **Support** | `support/MentalHealthSupportScreen` |

---

## 5. How the app talks to the backend

- **Base URL:** switches on `__DEV__` — **release** builds use `https://umatter-apcs.duckdns.org`
  (HTTPS via Caddy → gateway, cleartext forbidden), **dev** builds use `http://<PUBLIC_IP>:8080`.
  See [05-Deployment/04-DNS-HTTPS-and-Play-Release](../05-Deployment/04-DNS-HTTPS-and-Play-Release.md).
- **Auth:** stores the JWT in AsyncStorage; an axios interceptor attaches
  `Authorization: Bearer <token>` and handles 401 (re-login / refresh).
- **Push:** on login it registers its FCM token via `POST /api/v1/notification/api/v1/devices`
  (`{ profileId, deviceToken, platform: "ANDROID" }`); Notifee renders foreground notifications.
- **Chat:** opens a STOMP-over-WebSocket connection to the Social service for live messaging.
- **Video:** on `GET /api/v1/therapist/.../bookings/{id}/join` it receives the Zoom meeting number,
  password, and SDK JWT, then joins the call.
- **Media:** uploads (avatar/diary/treasure) go through the gateway; downloads use presigned
  `/mhsa-media/` URLs — now served over **HTTPS** (`S3_PUBLIC_ENDPOINT = https://umatter-apcs.duckdns.org`).

---

## 6. Configuration & docs in-repo
- `.env` — API base URL and keys.
- `thesis-mobile/docs/` — `8080_Gateway.md`, per-service controller references, `FCM_notification.md`.
- `thesis-mobile/README.md` — RN setup, run instructions.

---

## 7. Run it (dev)
```bash
cd thesis-mobile
npm ci
npm run start            # Metro bundler
npm run android          # build & launch on device/emulator
```
Point `BASE_URL` at a reachable backend (the Azure VM, or a local stack on your LAN).
