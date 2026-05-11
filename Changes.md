# Still In? – Bug Fix Log

## 🛠️ Your Fixes

### 1. Correct Status Code for Expired Tokens
- **The Problem**: When a JWT expires, the backend was returning a 500 Internal Server Error instead of a 401 Unauthorized. This prevented the frontend from identifying that the session had ended.
- **The Discovery**: Audited `server/middleware/auth.js`. The catch block was indiscriminately returning 500 for all verification errors, including `TokenExpiredError`.
- **The Fix**: Updated the middleware to check `err.name`. If it is `TokenExpiredError`, it now returns a 401 status with a clear "Token expired" message.

### 2. Type-Safe Duplicate Vote Prevention
- **The Problem**: Users could vote multiple times because the duplicate check was comparing a numeric `userId` against a string `email`.
- **The Discovery**: Code audit of `server/routes/poll.js`. Noticed `votedUserIds.find(id => id === req.user.email)` where `votedUserIds` stores IDs (numbers from `Date.now()`).
- **The Fix**: Changed the check to `votedUserIds.includes(userId)`, ensuring we check the actual ID stored in the array against the ID in the token.

### 3. Global Authentication Interceptor
- **The Problem**: The frontend didn't have a centralized way to handle 401 errors. If a session expired, the UI stayed on the dashboard while background requests failed silently.
- **The Discovery**: Noticed the lack of a response interceptor in `client/src/api/client.js`.
- **The Fix**: Added an Axios response interceptor that watches for 401 status codes. When detected, it clears `localStorage` and forces a redirect to the login page.

### 4. Stopping Zombie Polling
- **The Problem**: The dashboard polling interval (`setInterval`) continued to fire every 10 seconds even after the session expired, leading to a resource leak and noisy errors.
- **The Discovery**: Observed the Network tab after token expiry (60s). Requests to `/api/poll` kept firing every 10 seconds.
- **The Fix**: Updated `client/src/pages/Dashboard.jsx` to catch 401 errors in the `fetchPoll` function. When an auth error occurs, it calls `clearInterval(intervalRef.current)` and triggers `logout()` to clean up the state and redirect.

---

## 📈 Verification Results
- **API Response**: After 60s, `/api/poll` returns **401 Unauthorized**.
- **UI Behavior**: The app immediately redirects to the **Login page** once the first 401 is received.
- **Polling**: No further requests are seen in the Network tab after the redirect.
- **Voting**: Attempting to vote twice now correctly shows "You have already voted!".
