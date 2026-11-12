# Research: camera doorbells on Echo Show

Research status: 2026-11-12. This document is the evidence base for the Doorbridge design and backlog.

**Markers:** ✅ = backed by a source · 💭 = our own inference, not backed by a source · ❓ = open, must be settled by testing.

## Question

Can a camera doorbell that does not come from Amazon/Ring (example: UniFi Protect) be connected to Alexa so that it behaves like a Ring doorbell?

- chime or announcement on selected Echo devices when the button is pressed
- automatic live view on the Echo Show when the button is pressed
- live view on demand ("Alexa, show the front door")
- optionally, the image of a second camera covering the entrance instead of the doorbell camera

Constraints: video processing self-hosted in a Docker container, publishable as open source, and a Skill Store release only if it costs an open-source developer nothing.

## Summary

1. A **Smart Home skill** (not a custom skill) with the interfaces `Alexa.DoorbellEventSource` and `Alexa.RTCSessionController` is required. Only both together produce the Ring-like behaviour. ✅
2. Both interfaces are available for **de-DE**. ✅
3. The skill endpoint must be an **AWS Lambda** function (Europe: Ireland, eu-west-1). It can act as a thin relay to a self-hosted backend. ✅ / 💭
4. For personal use, a skill in **development mode** on one's own Amazon account is enough. No certification, Lambda within the always-free tier. ✅
5. A **public Skill Store release** requires a permanently operated, multi-tenant cloud service with a test account. That conflicts with the "no cost" constraint. The optional Works with Alexa (WWA) certification charges lab fees. ✅ / 💭
6. **Recommendation:** the "Bring Your Own Skill" model. Every user creates their own skill and Lambda function; the project ships code, container and guide. The architecture is cut so that a public skill could be added later. 💭

## 1 Alexa interfaces

### Alexa.DoorbellEventSource (ring)

- On every press the backend sends a `DoorbellPress` event to the Alexa Event Gateway. The user chooses in the Alexa app which Echo devices react. ✅ [1]
- If the device also implements `Alexa.RTCSessionController` and an Echo Show is selected, the Show plays the announcement and automatically shows the live view of the doorbell camera. Without RTCSessionController there is only the announcement. ✅ [1]
- Display categories: `DOORBELL`; for a video doorbell `CAMERA` and `DOORBELL`. ✅ [1]
- After a `DoorbellPress` you must wait **at least 30 seconds** before sending the next one for the same device. ✅ [1]
- No voice commands, no directives. `Alexa.EndpointHealth` is recommended in addition. ✅ [1]

### Alexa.RTCSessionController (live view via WebRTC)

- Flow: `InitiateSessionWithOffer` → `AnswerGenerated` → `SessionConnected` → `SessionDisconnected`. ✅ [2][3]
- An SDP answer must be returned **within 6 seconds** of an SDP offer, and the device must be able to stream for at least **1 minute** continuously. ✅ [3]
- Video: **H.264** up to profile High / level 4.1, **480p to 1080p**. Audio: Opus (preferred) or PCMU/PCMA. ✅ [3]
- ICE: UDP or TCP, but **IPv4 only**. **No trickle ICE**; all candidates belong in the SDP answer. Without a public gateway, STUN or TURN candidates must be included. BUNDLE and rtcp-mux are mandatory, plus NACK, PLI, FIR and REMB. ✅ [3]
- Echo Show: video is one-way only. Audio is one-way or two-way. Half duplex via `isFullDuplexAudioSupported: false` when there is no echo cancellation. ✅ [3]
- Built-in German utterances exist, e.g. "Alexa, zeige mir die Haustürkamera", "Alexa, sprich mit der Haustür", "Alexa, geh nach Hause". ✅ [2]
- Amazon names RTCSessionController as the preferred interface for security cameras. It is mandatory for WWA; `Alexa.CameraStreamController` no longer counts there. ✅ [4][5]
- After a doorbell event the automatic live view runs for **at most 15 seconds**. Touching the screen or activating the microphone extends the session to up to 10 minutes. ✅ [6]
- The Smart Home Debugger in the Developer Console logs WebRTC sessions live and can be used for testing. ✅ [7]
- ❓ The documentation does not state whether media flows directly between the Echo and the device or through the Amazon cloud. To be settled by testing (relevant for the TURN question).

### Language support

`Alexa.DoorbellEventSource` and `Alexa.RTCSessionController` both list **de-DE** among their supported languages. ✅ [8]

### Showing a second camera on ring

💭 There is no API option "show camera X instead". Alexa always requests the WebRTC stream of the doorbell endpoint. The backend decides which source it serves behind it, so two approaches are possible:

- switch the source: the doorbell endpoint serves the entrance camera
- composition: both cameras side by side or picture-in-picture (transcoding, CPU/GPU load)

## 2 Skill infrastructure

### Endpoint: AWS Lambda

- Smart Home skills use an AWS Lambda function as their endpoint, configured by ARN. ✅ [9][10]
- Region for de-DE: **EU (Ireland)**. Event Gateway Europe: `https://api.eu.amazonalexa.com/v3/events`. ✅ [11][12]
- Alexa-hosted skills do not work for Smart Home; they support the custom model only. ✅ [13]
- Home Assistant shows the pattern: the Lambda function only forwards to the user's own instance, which must be reachable via HTTPS on port 443. ✅ [14] The same pattern fits here. 💭

### Account linking

- Mandatory for Smart Home skills: OAuth 2.0 with the authorization code grant. ✅ [4][15]
- 💭 For self-hosting: a small OAuth server inside the bridge container, or an existing OIDC provider (e.g. Authentik, Keycloak).

### Proactive events (sending the ring)

- After account linking, Alexa sends the directive `Alexa.Authorization.AcceptGrant` with an authorization code. It is exchanged at Login with Amazon (LWA) for an access and a refresh token. ✅ [16][17]
- Access tokens expire after about **60 minutes** and must be refreshed; otherwise the gateway answers with 401. ✅ [16][18]
- The skill must have the "Send Alexa Events" permission enabled, and the Lambda function must handle `AcceptGrant`, otherwise account linking fails. ✅ [19]
- The token goes into the `Authorization` header and additionally into `endpoint.scope` of the message. ✅ [17]

### Costs

- **AWS Lambda:** always-free tier, 1 million requests and 400,000 GB-seconds per month, for all accounts without expiry. ✅ [20][21] A relay function stays free for private use. 💭
- **CloudWatch Logs** are not included but have their own free tier (including 5 GB ingestion per month). ✅ [20] Keep log retention short. 💭
- **AWS Kinesis Video Streams** (used in the Amazon sample) is not needed, because the SDP is part of the directive and can be passed to go2rtc. 💭 [6]
- **Amazon Developer account:** free. ✅ [22]

## 3 Publishing and certification

| Path | What is needed | Cost | Assessment |
|------|----------------|------|------------|
| Skill in development mode (per user) | own developer and AWS account, own Lambda function | free within the free tier ✅ | **recommended** 💭 |
| Public skill in the Alexa Skills Store | skill certification, test account with account linking and at least one device reachable around the clock, privacy policy, customer instructions ✅ [23][24] | the sources name no fee for skill certification 💭, but a permanently operated multi-tenant relay service and support are needed | not operable free of charge 💭 |
| Works with Alexa (WWA) | certification by external test labs, programme for device makers | lab fee per submission ✅ [25][26] | excluded |

Further points for a later release:

- Use Amazon marks referentially only ("… for Amazon Alexa", not "Alexa skill for …"). "Alexa" does not belong in a subdomain or domain root. ✅ [27]
- Third-party marks (e.g. UniFi) infringe third-party rights unless permission is shown. ✅ [28] Hence a neutral project name. 💭
- Under Alexa+ the documentation now calls skills "add-ons"; the Smart Home interfaces stay the same. ✅ [1][2]

## 4 Video source and processing (self-hosted)

- **go2rtc** accepts a raw SDP offer (`application/sdp` or JSON) via `POST /api/webrtc?src=<stream>` and returns the SDP answer. It can therefore serve the Alexa offer directly. ✅ [29][49]
- 💭 The critical part is the set of Alexa constraints go2rtc has to meet: no trickle ICE, IPv4 only, few candidates, H.264 within level 4.1. This is the first technical spike.
- 💭 Doorbell cameras often deliver H.264 in portrait or high resolution. Use a substream (≤ 1080p) or transcode with ffmpeg if needed.
- 💭 TURN: for an Echo on the same network, host candidates are probably enough. For the Alexa app on the go, STUN/TURN is needed (self-hosted, e.g. coturn) ❓.

## 5 Ring event from the source

- UniFi Protect offers an official **Integration API** (HTTPS and WebSocket, bearer token / API key). ✅ [30]
- According to the Home Assistant documentation, the **doorbell ring event still requires the private API** via a local user. The public API is being extended. ✅ [31]
- Libraries: `uiprotect` (Python) is migrating to the public API and no longer accepts new features based on the private API ✅ [32]. `unifi-protect` (Node) provides a typed event stream including rings ✅ [33]. `ClifHouck/unified` (Go) exposes Protect WebSocket streams as Go channels ✅ [34].
- Since 2026 Home Assistant unifies doorbell events on the standard event type `ring` (`DoorbellEventType.RING`). Integrations without it will stop working from HA 2027.4. ✅ [35]
- 💭 Conclusion: inputs as plugins (Protect directly, Home Assistant, generic webhook, MQTT) so the project is not tied to one vendor.

## 6 Existing solutions (scope)

- **Home Assistant (Alexa Smart Home):** doorbell announcements work; HA reports `DoorbellEventSource` actively ✅ [36]. Forum users can call up the camera image by voice, but not show it automatically from an automation ✅ [37]. ❓ Suspected cause: camera and doorbell are separate endpoints, without RTCSessionController.
- **Scrypted:** the Alexa plugin streams cameras to the Echo Show but only works through the Scrypted Cloud ✅ [38]. ❓ Automatic live view on ring not verified.
- **Amazon sample** `alexa-sh-camera-webrtc`: video doorbell with both interfaces, signalling via AWS KVS. Good reference for discovery and events ✅ [6]

### RTSP solutions via CameraStreamController (e.g. Monocle)

- Monocle uses the Alexa interface `CameraStreamController` and can only serve regions and devices for which Amazon offers this interface. Alexa devices only play **RTSP** in this case (H.264, audio AAC or G.711). ✅ [39]
- The Monocle Gateway only tunnels or proxies the RTSP stream TLS-encrypted on port 443. It does not read or decode the stream; the Echo device does the decoding. ✅ [40][41]
- The website mentions neither doorbell events nor two-way audio nor automatic live view, only on-demand viewing by voice. ✅ [42]
- 💭 This explains why ring events and talkback do not work with such solutions: `CameraStreamController` is a playback-only interface. Two-way audio exists only with `RTCSessionController` (see section 1), and `CameraStreamController` no longer counts for WWA. Doorbridge therefore replaces such a setup rather than complementing it.

### Two-way audio (talkback)

- Alexa supports one-way or two-way audio on the Echo Show. Full duplex requires echo cancellation; otherwise half duplex (push-to-talk). ✅ [3]
- UniFi Protect offers **talkback sessions** through the public Integration API; the openHAB binding uses them alongside RTSPS and WebRTC streams. ✅ [43]
- The public API returns a talkback stream URL including the audio configuration via `create_talkback_session_public`. ✅ [44] The channel is a send-only WebSocket: Protect accepts the audio and plays it on the speaker without feedback or codec negotiation. ✅ [45]
- Protect doorbells with talkback are full duplex. ✅ [46] homebridge-unifi-protect lacks echo cancellation, though: the Home app hears an echo, the sound at the door is clean. ✅ [47]
- go2rtc supports two-way audio only for certain sources (among them doorbird, dvrip, exec, isapi, ring, rtsp with ONVIF backchannel, tapo, tuya, webrtc, wyze, xiaomi). **UniFi Protect is not among them.** ✅ [48]
- 💭 Conclusion: the bridge implements the back channel itself. It takes the Echo Show's audio track (Opus/PCMU) from the WebRTC session, converts it into the format the talkback session dictates, and writes it into the WebSocket. In ADR 3 this argues for terminating WebRTC with pion in the bridge rather than only passing through go2rtc.
- 💭 Start with half duplex (push-to-talk, `isFullDuplexAudioSupported: false`), because without echo cancellation the echo would be sent back to the Echo Show. Full duplex only after a test. ❓

## Sources

1. [Alexa.DoorbellEventSource Interface 3](https://developer.amazon.com/docs/device-apis/alexa-doorbelleventsource.html)
2. [Alexa.RTCSessionController Interface 3](https://developer.amazon.com/docs/alexaplus/device-apis/alexa-rtcsessioncontroller.html)
3. [About the Real-Time Communication Interface](https://developer.amazon.com/docs/device-apis/about-alexa-rtc.html)
4. [Understand Smart Home Security Skills](https://developer.amazon.com/es-ES/docs/alexa/device-apis/overview-smart-home-security.html)
5. [Smart Home Skill APIs](https://developer.amazon.com/de-DE/docs/alexa/device-apis/smart-home-general-apis.html)
6. [Sample repository alexa-sh-camera-webrtc](https://github.com/clevertonh/alexa-sh-camera-webrtc)
7. [Smart Home Debugger for WebRTC Skills](https://developer.amazon.com/docs/smarthome/test-live-debug-audio-video.html)
8. [List of Alexa Interfaces and Supported Languages](https://developer.amazon.com/it/docs/device-apis/list-of-interfaces.html)
9. [Step 3: Configure the Service Endpoint](https://developer.amazon.com/de-DE/docs/alexa/smarthome/configure-endpoint-tutorial.html)
10. [Implement Smart Home Skill Code](https://developer.amazon.com/docs/smarthome/implement-your-skill.html)
11. [Develop Smart Home Skills for Multiple Languages](https://developer.amazon.com/de-DE/docs/alexa/smarthome/develop-smart-home-skills-in-multiple-languages.html)
12. [Develop Smart Home Add-ons for Multiple Languages](https://developer.amazon.com/docs/alexa/smarthome/develop-smart-home-skills-in-multiple-languages.html)
13. [About Alexa-Hosted Skills](https://developer.amazon.com/de-DE/docs/alexa/hosted-skills/build-a-skill-end-to-end-using-an-alexa-hosted-skill.html)
14. [Home Assistant: Amazon Alexa Smart Home Skill](https://www.home-assistant.io/integrations/alexa.smart_home/)
15. [Smart Home and Video Skills Certification Guide](https://developer.amazon.com/docs/smarthome/smart-home-skill-publishing-guide.html)
16. [Request Access to the Alexa Event Gateway](https://developer.amazon.com/de-DE/docs/alexa/smarthome/authenticate-a-customer-permissions.html)
17. [Request Access to the Alexa Event Gateway (token usage)](https://developer.amazon.com/de/docs/smarthome/authenticate-a-customer-permissions.html)
18. [Refresh tokens before expiration](https://developer.amazon.com/en-US/docs/alexa/smarthome/authenticate-a-customer-permissions.html)
19. [Gist: Home Assistant Lambda with AcceptGrant note](https://gist.github.com/awarecan/630510a9742f5f8901b5ab284c25e912)
20. [AWS Lambda cost breakdown 2026 (Wiz)](https://wiz.io/academy/cloud-cost/aws-lambda-cost-breakdown)
21. [AWS Free Tier Serverless](https://aws.amazon.com/free/serverless)
22. [Works with Alexa Program (developer account is free)](https://developer.amazon.com/en-US/alexa/connected-devices/launch/works-with-alexa)
23. [Smart Home and Video Skills Certification Guide (test account)](https://developer.amazon.com/de-DE/docs/alexa/smarthome/smart-home-skill-publishing-guide.html)
24. [Troubleshooting Certification Failures in Smart Home](https://developer.amazon.com/docs/alexa/certify-skills/troubleshoot-smart-home-skill-failures.html)
25. [Works with Alexa FAQ](https://developer.amazon.com/docs/smarthome/wwa-faq.html)
26. [Works with Alexa Overview](https://developer.amazon.com/docs/smarthome/wwa-overview.html)
27. [Policy Certification Tips on Amazon IP](https://developer.amazon.com/en-US/blogs/alexa/alexa-skills-kit/2020/08/Policy-Certification-Tips-on-Amazon-IP-and-ISP)
28. [Policy Requirements for Alexa Skills](https://developer.amazon.com/fr-FR/docs/alexa/custom-skills/policy-requirements-for-an-alexa-skill.html)
29. [go2rtc API (Context7)](https://context7.com/skrashevich/go2rtc)
30. [openHAB UniFi Protect binding (Integration API)](https://v51.openhab.org/addons/bindings/unifiprotect/)
31. [Home Assistant: UniFi Protect](https://www.home-assistant.io/integrations/unifiprotect)
32. [uiprotect on PyPI](https://pypi.org/project/uiprotect/13.4.0/)
33. [unifi-protect (npm) README](https://cdn.jsdelivr.net/npm/unifi-protect@5.3.1/README.md)
34. [ClifHouck/unified](https://github.com/ClifHouck/unified)
35. [HA Developer Blog: Doorbell standard event type](https://developers.home-assistant.io/blog/2026/04/15/doorbell-standard-event-type)
36. [HA Core: alexa/state_report.py](https://git.sudo.is/home-assistant/core/src/branch/dev/homeassistant/components/alexa/state_report.py)
37. [HA forum: Ubiquiti cameras and Echo Show 8](https://community.home-assistant.io/t/ubiquiti-cameras-and-echo-show-8/341802)
38. [Scrypted Alexa documentation](https://docs.scrypted.app/alexa.html)
39. [Monocle: Additional Tips & Tricks](https://monoclecam.com/tips)
40. [Monocle Gateway](https://monoclecam.com/gateway)
41. [Monocle forum: Alexa Suddenly Stopped with Unifi](https://forum.monoclecam.com/post/4272)
42. [Monocle home page](https://monoclecam.com/)
43. [openHAB UniFi Protect binding (talkback)](https://www.openhab.org/addons/bindings/unifi/doc/protect)
44. [uiprotect API: create_talkback_session_public](https://uiprotect.readthedocs.io/en/latest/api/)
45. [unifi-protect: talkback-session.ts](https://cdn.jsdelivr.net/npm/unifi-protect@5.0.0/src/transport/talkback-session.ts)
46. [homebridge-unifi-protect: Doorbell.md](https://cdn.jsdelivr.net/npm/homebridge-unifi-protect2@3.6.3/docs/Doorbell.md)
47. [homebridge-unifi-protect: FeatureOptions.md](https://cdn.jsdelivr.net/npm/homebridge-unifi-protect2@3.6.3/docs/FeatureOptions.md)
48. [go2rtc: Two-way audio (Docsearch)](https://docsearch.algolia.com/mcp/docs/repo/alexxit/go2rtc)

### Additional sources (added during project setup, 2026-11-12)

49. [go2rtc OpenAPI: `/api/webrtc?src={src}`](https://github.com/AlexxIT/go2rtc/blob/master/website/api/openapi.yaml)
50. [go2rtc WebRTC module: candidates and ice_servers](https://github.com/AlexxIT/go2rtc/blob/master/internal/webrtc/README.md)
51. [go2rtc repository (MIT licence)](https://github.com/AlexxIT/go2rtc)
52. [pion/webrtc (MIT licence)](https://github.com/pion/webrtc)
53. [AWS Lambda: Building with Go (`provided.al2023`)](https://docs.aws.amazon.com/lambda/latest/dg/lambda-golang.html)
54. [AWS Lambda: Configure function timeout](https://docs.aws.amazon.com/lambda/latest/dg/configuration-timeout.html)
55. [CloudWatch Logs: log groups and retention](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/Working-with-log-groups-and-streams.html)
56. [coturn TURN server](https://github.com/coturn/coturn)
57. [Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0)
58. [Apache: Applying the Apache License (NOTICE file)](https://infra.apache.org/apply-license.html)
59. [Google open source: AGPL policy](https://opensource.google/documentation/reference/using/agpl-policy)
60. [Developer Certificate of Origin](https://developercertificate.org/)
61. [DCO GitHub App](https://github.com/dcoapp/app)
62. [Conventional Commits 1.0.0](https://www.conventionalcommits.org/en/v1.0.0/)
63. [semantic-release](https://semantic-release.gitbook.io/semantic-release/)
64. [GitHub Actions: events that trigger workflows (`schedule`, `workflow_dispatch`)](https://docs.github.com/en/actions/writing-workflows/choosing-when-your-workflow-runs/events-that-trigger-workflows)
65. [MegaLinter](https://megalinter.io/latest/)
66. [Docker: multi-platform builds](https://docs.docker.com/build/building/multi-platform/)
67. [Docker: build attestations (`--provenance`, `--sbom`)](https://docs.docker.com/build/metadata/attestations/)
68. [docker/metadata-action (semver tags)](https://github.com/docker/metadata-action)
69. [Renovate documentation](https://docs.renovatebot.com/)
70. [GitHub Container Registry](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)
71. [RFC 6749: OAuth 2.0 (authorization code grant, section 4.1)](https://datatracker.ietf.org/doc/html/rfc6749#section-4.1)
72. [RFC 2104: HMAC](https://datatracker.ietf.org/doc/html/rfc2104)
73. [OWASP Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html)
74. [OWASP Threat Modeling Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Threat_Modeling_Cheat_Sheet.html)
75. [FFmpeg filters (`hstack`, `overlay`)](https://ffmpeg.org/ffmpeg-filters.html)
76. [Compose file reference: services (`healthcheck`)](https://docs.docker.com/reference/compose-file/services/)
77. [MQTT](https://mqtt.org/)
