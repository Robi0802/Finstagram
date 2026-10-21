# FinstaGram — Product and Implementation Plan

## Purpose

This document records the agreed v1 product specification and an implementation sequence for future work. It is planning material; implementation will happen only when requested.

## Product summary

FinstaGram is an adults-focused, friend-oriented social app. The first release target is an invited iOS TestFlight beta. The app is free, uses a clean, minimal, mobile-first interface, and follows the iOS system appearance.

Users must sign in before browsing. Profiles and posts are public to signed-in users. V1 is centered on sharing and a chronological feed from followed accounts.

## V1 scope

### Main navigation

The app has four main tabs:

1. **Home** — chronological feed of posts from followed users.
2. **Messages** — one-to-one conversations with text and image messages.
3. **Explore** — search for people and browse recent public posts.
4. **Profile** — view the signed-in user's profile and posts; other profiles are reachable from posts and search.

Post composition is entered from an action in Home so the four-tab navigation remains unchanged.

### User capabilities

- Sign up and sign in with Apple or Google.
- Set a unique username and display name during onboarding. Bio and avatar are optional.
- Receive popular/recent user suggestions after profile setup; contact access is not required.
- Follow and unfollow users. Profiles are public, so follows do not require approval.
- Create a post with one photo or one video, with a caption. Videos may be up to 60 seconds.
- View the followed-user feed in reverse chronological order.
- Like posts, add comments, and delete one's own comments.
- Edit the caption of one's own post, delete one's own post, and view one's posts on their profile.
- Start a one-to-one direct message with any signed-in user; messages may contain text or an image.
- Receive live updates for social activity and messages while the app is open.

### Out of scope for v1

- Android and web releases.
- Private profiles or follow requests.
- Group chats, message editing/deletion, read receipts, typing indicators, and push notifications.
- Block/report tools and self-serve account deletion.
- Monetization, contact matching, and in-app camera capture.
- Multiple media items in one post, ranked feeds, and end-to-end encrypted messaging.

## Core user journeys

### First-time user

1. Open the app and see the branded sign-in screen.
2. Continue with Apple or Google.
3. Choose a unique username and display name; optionally add a bio and avatar.
4. Choose from popular/recent account suggestions to follow.
5. Land on Home. If the user follows nobody or the feed has no posts, show a useful empty state that links to Explore/search.

### Returning user

1. Clerk restores the signed-in session.
2. The user lands on the tab they last used or Home (implementation default: Home).
3. Convex supplies the current profile and live data for the active screen.

### Posting and interacting

1. Start composition from Home.
2. Select a photo or video from the system photo library, add a caption, and publish.
3. Show upload progress and recoverable failure feedback; do not show a post as published until its media and record are saved.
4. From Home, Explore, or a profile, open a post, like it, comment, edit its own caption, or delete the post/comment if the signed-in user owns it.

### Messaging

1. Open a profile or conversation list and start a one-to-one conversation with any signed-in user.
2. Send text or select and upload an image.
3. Show new messages in the open conversation through Convex live updates. Push notifications are not part of v1.

## Data model and ownership

Clerk is the source of truth for authentication identity. Convex is the source of truth for app profiles, social relationships, posts, comments, likes, conversations, messages, and uploaded-file references. Backend functions must enforce authentication, ownership, and conversation membership; client-side checks alone are not authorization.

Conceptual Convex records:

- **Profiles:** Clerk user ID, normalized unique username, display name, optional bio/avatar reference, creation/update timestamps.
- **Posts:** author profile, caption, media type, Convex storage ID, creation/update timestamps.
- **Follows:** follower, followed user, creation timestamp; one record per directed pair.
- **Likes:** user and post; one record per user/post pair.
- **Comments:** author, post, text, creation timestamp.
- **Conversations:** the two participant profile IDs and creation/last-activity timestamps; one conversation per participant pair.
- **Messages:** conversation, sender, text or image storage reference, creation timestamp.

Username characters are limited to letters, numbers, underscores, and periods. Usernames must be unique and changeable. Reads and writes to conversations are limited to their participants. Post deletion removes the post, its comments, and its stored media.

## Architecture and technical constraints

- **Client:** Existing Expo / React Native app, iOS first, using Expo Router file-based routing and four native tabs.
- **Styling:** NativeWind, with light and dark appearances following the iOS system setting.
- **Authentication:** Clerk owns sign-in and session identity. Use custom branded screens with Apple and Google sign-in.
- **Backend/database:** Convex owns application data and real-time queries/mutations.
- **Media:** Use Convex File Storage for post and chat uploads.
- **No additional backend service** is currently selected.

The repository currently uses Expo SDK `~57.0.31`, Expo Router `~57.0.29`, React Native `0.86.3`, and React `19.2.3`. For Expo SDK 57, Expo documents native tabs under `expo-router/unstable-native-tabs`; this API is explicitly unstable. NativeWind's current installation documentation identifies v4.2.7 as adding SDK 57 support. Clerk's current Expo guide targets `@clerk/expo` 4.x and requires a development build for its native Apple/Google sign-in hooks. Verify exact compatible package versions and read the matching Expo versioned documentation before implementation; do not assume unversioned or newer-SDK examples apply.

A development build is required for native sign-in modules. Configure Clerk Apple/Google provider credentials, app identifiers, and required Expo config plugins before device testing. Keep secrets out of client-exposed environment variables; only public client keys belong in the app bundle.

## State, failure behavior, and security

- Convex reactive queries provide live in-app updates. Feed and long lists should be paginated rather than loading all records at once.
- Show loading, empty, retry, and error states for feeds, search, profiles, conversations, and media uploads.
- Handle cancelled sign-in without treating it as an error; show actionable feedback for provider/network failures.
- Prevent duplicate follows, likes, and conversations through backend constraints/logic, and make repeated user actions safe.
- Validate ownership on post/comment deletion and caption edits, and participant membership for every conversation/message operation.
- Treat DMs as private server-stored messages accessible only to conversation participants in the app. They are not end-to-end encrypted.
- Do not log message bodies, captions, or other private content in operational logs.
- Post deletion must remove related comments and media. Also remove related likes to prevent orphaned interaction records (assumption to confirm).
- If media upload succeeds but saving the post/message fails, clean up the unattached upload where feasible and let the user retry without creating duplicates.

## Quality and release expectations

V1 is ready for invited TestFlight use when:

- Apple and Google sign-in, session restoration, onboarding, and sign-out work on an iPhone development/production build.
- Users can create/edit/delete their own posts and comments, and cannot modify another user's content.
- Following, chronological feeds, Explore search/recent posts, likes, comments, and profile views work with empty and populated data.
- One-to-one text/image messaging updates live and is inaccessible to non-participants.
- Upload, cancellation, poor-network, and service-error states provide clear feedback and recover safely.
- Automated checks cover important backend rules and core UI behavior; lint and TypeScript checks pass; core journeys are manually checked on an iPhone.
- TestFlight uses production-like Clerk and Convex configuration distinct from local development.

No fixed release date or service budget has been set. Basic error reporting and service health visibility are assumed; provider/tool choice can be made during implementation.

## Step-by-step implementation roadmap

### 1. Foundation and compatibility

- Recheck the Expo SDK 57 versioned documentation and package compatibility.
- Configure NativeWind and its required Metro/Babel/Tailwind setup for the installed SDK.
- Add the four Expo Router native tabs and stack navigation for detail screens.
- Establish local development environment variables and a development build workflow.
- **Exit criteria:** App starts on iOS, tabs and nested screens navigate, NativeWind styles render in light and dark appearances.

### 2. Authentication and onboarding

- Add Clerk Expo provider/token persistence and Convex Clerk authentication.
- Configure Apple and Google providers, app identifiers, native modules, and development-build plugins.
- Build branded sign-in screens and authenticated/unauthenticated route handling.
- Create profile onboarding, normalized unique username checks, and popular/recent follow suggestions.
- **Exit criteria:** New and returning users authenticate, complete onboarding once, and reach the app with valid Convex identity.

### 3. Profiles, search, and follow graph

- Implement profile read/edit flows and user search in Explore.
- Implement follow/unfollow and onboarding suggestions.
- Enforce unique directed follow relationships and authenticated access in Convex.
- **Exit criteria:** Profiles/search render correctly; follow state is consistent across profile, Explore, and Home.

### 4. Posts, media, feed, and interactions

- Add system photo-library selection and Convex upload flows for images and videos.
- Implement post creation, paginated chronological feed, recent-post Explore grid, likes, comments, caption edits, and deletion.
- Implement cleanup of associated records and Convex media on post deletion.
- **Exit criteria:** Media and post actions work on device; ownership rules hold; upload errors and empty states are handled.

### 5. Direct messaging

- Implement one-to-one conversation creation/deduplication and participant-only queries/mutations.
- Add text and image messages, upload status, conversation list, and live message updates.
- **Exit criteria:** Both participants see live updates; other users cannot access the conversation; retry does not duplicate messages.

### 6. Hardening and TestFlight beta

- Complete automated backend/UI coverage and on-device end-to-end verification.
- Resolve launch-blocking privacy, media-limit, and account-deletion decisions.
- Configure production Clerk/Convex environments, signing, and TestFlight distribution.
- **Exit criteria:** Quality expectations above are met and beta configuration is documented.

## Assumptions to confirm during implementation

- Username uniqueness is case-insensitive after normalization.
- Avatar and post media are represented by Convex storage references; avatar upload behavior is included even though avatar is optional.
- Post deletion also deletes likes associated with that post.
- The Home tab is the default destination after sign-in and after app relaunch.
- Composition uses a Home action rather than an additional tab.
- No email/password fallback is enabled; Apple and Google are the launch providers.
- The default suggestion ordering can use a simple popular/recent heuristic, not a personalized ranking system.

## Open risks and decisions

- **Safety:** Anyone signed in can start a DM, but v1 has no block/report controls. Decide whether this is acceptable before a public launch.
- **Account deletion:** Self-serve deletion is deferred. Define a deletion/support process before public release.
- **Media limits:** Set image and video file-size limits, supported formats, and upload constraints. Convex upload requests have a two-minute timeout; confirm that expected 60-second videos fit the chosen limits and network conditions.
- **Video delivery:** Validate playback performance, transcoding needs, bandwidth, and storage costs using Convex File Storage.
- **Text limits:** Define maximum caption, comment, and message lengths and username validation details.
- **Privacy policy and age policy:** Confirm adult-only positioning, age handling, retention, and user-facing privacy terms before distribution.
- **Scale and cost:** Establish expected beta cohort size and monitor Convex storage, bandwidth, and query usage as real usage becomes measurable.
- **Observability:** Select an error-reporting/analytics approach that avoids collecting private content and meets the app's privacy expectations.
