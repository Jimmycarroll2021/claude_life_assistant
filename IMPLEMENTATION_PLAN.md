# Life Assistant Mobile App - Implementation Plan (REVISED)

## Overview
Build a mobile app with clean UI that acts as your personal coach, powered by **Claude Code CLI** (using your Max plan), reading from CLAUDE.md/NOW.md files in this repo.

**Key Change**: Uses Claude Code CLI instead of Claude API = **$0/month** instead of $50-150/month.

---

## Architecture

### System Components

```
┌─────────────────────────────────────────┐
│    Mobile Interface (PWA or App)        │
│  ┌─────────────────────────────────┐   │
│  │  Chat Interface                  │   │
│  │  Quick Actions (/start-day, etc)│   │
│  │  Settings & Reminders            │   │
│  │  Push Notification Handler       │   │
│  └─────────────────────────────────┘   │
└──────────────┬──────────────────────────┘
               │ HTTPS
               │
┌──────────────▼──────────────────────────┐
│    Backend API (Node.js/Express)        │
│  ┌─────────────────────────────────┐   │
│  │  Wraps Claude Code CLI           │   │
│  │  /api/chat → `claude` command    │   │
│  │  /api/slash-command              │   │
│  │  /api/reminders                  │   │
│  └─────────────────────────────────┘   │
└──────────────┬──────────────────────────┘
               │
        ┌──────┴──────────┐
        │                 │
┌───────▼──────────┐  ┌──▼────────────┐
│ Claude Code CLI  │  │ This Git Repo │
│ (Max Plan)       │  │ CLAUDE.md     │
│ Existing cmds:   │  │ NOW.md        │
│ /start-day       │  │ .claude/      │
│ /end-day         │  │  commands/    │
│ /check-day       │  │               │
└──────────────────┘  └───────────────┘
```

---

## Part 1: Backend API

### Tech Stack
- **Runtime**: Node.js 20+
- **Framework**: Express.js
- **Claude Integration**: `child_process` wrapping `claude` CLI
- **Session Management**: In-memory Map (simple) or Redis (production)
- **Scheduler**: node-cron
- **Push Notifications**: Firebase Admin SDK (or simple polling)
- **Auth**: API Key (single user)
- **CORS**: For PWA access from phone

### Project Structure
```
backend/
├── src/
│   ├── server.js              # Main entry point
│   ├── config/
│   │   ├── claude.js          # Claude API config
│   │   ├── firebase.js        # Firebase config
│   │   └── env.js             # Environment variables
│   ├── controllers/
│   │   ├── chatController.js  # Handle chat requests
│   │   ├── fileController.js  # CRUD for NOW.md
│   │   └── reminderController.js
│   ├── services/
│   │   ├── claudeService.js   # Claude API wrapper
│   │   ├── contextService.js  # Read CLAUDE.md + NOW.md
│   │   ├── notificationService.js
│   │   └── schedulerService.js
│   ├── middleware/
│   │   ├── auth.js            # API key validation
│   │   └── errorHandler.js
│   └── utils/
│       ├── logger.js
│       └── validators.js
├── .env
├── package.json
└── ecosystem.config.js        # PM2 config for Pi5
```

### API Endpoints

#### 1. POST /api/chat
```json
Request:
{
  "message": "Should I take this freelance project?",
  "userId": "user_123"
}

Response:
{
  "reply": "Is this aligned with your mission or just money?",
  "sessionId": "sess_abc",
  "timestamp": "2025-12-29T10:30:00Z"
}
```

#### 2. GET /api/context
```json
Response:
{
  "claudeMd": "# CLAUDE.md content...",
  "nowMd": "# NOW.md content...",
  "lastUpdated": "2025-12-29T08:00:00Z"
}
```

#### 3. POST /api/update-now
```json
Request:
{
  "updates": {
    "currentMIT": "Ship mobile app backend",
    "blockers": ["Need to test push notifications"]
  }
}

Response:
{
  "success": true,
  "updatedAt": "2025-12-29T10:30:00Z"
}
```

#### 4. GET/POST /api/reminders
```json
POST Request:
{
  "time": "08:00",
  "message": "Morning check-in",
  "frequency": "daily"
}

GET Response:
{
  "reminders": [
    {
      "id": "rem_1",
      "time": "08:00",
      "message": "Morning check-in",
      "frequency": "daily",
      "enabled": true
    }
  ]
}
```

#### 5. POST /api/slash-command
```json
Request:
{
  "command": "start-day"
}

Response:
{
  "prompt": "Morning kickoff. Sets intentions and MIT for the day.",
  "response": "What's your MIT today?"
}
```

### Claude Code CLI Integration Strategy

**How It Works**:
Claude Code CLI automatically reads CLAUDE.md + NOW.md from the current directory. We leverage this by running `claude` commands from this repo directory.

**Basic Wrapper**:
```javascript
const { spawn } = require('child_process');
const path = require('path');

async function askClaude(message, sessionId = null) {
  return new Promise((resolve, reject) => {
    const repoPath = '/home/user/claude_life_assistant';

    const claude = spawn('claude', [], {
      cwd: repoPath, // Run from repo directory
      stdio: ['pipe', 'pipe', 'pipe']
    });

    let output = '';
    let errorOutput = '';

    // Send message to Claude
    claude.stdin.write(message + '\n');
    claude.stdin.end();

    // Collect response
    claude.stdout.on('data', (data) => {
      output += data.toString();
    });

    claude.stderr.on('data', (data) => {
      errorOutput += data.toString();
    });

    claude.on('close', (code) => {
      if (code === 0) {
        resolve(output.trim());
      } else {
        reject(new Error(errorOutput));
      }
    });
  });
}
```

**Slash Commands**:
```javascript
// Existing slash commands already work
async function runSlashCommand(command) {
  // Commands: start-day, end-day, check-day
  return askClaude(`/${command}`);
}
```

**Conversation State**:
- Claude CLI maintains session state automatically
- Each API request = new Claude CLI instance
- For multi-turn: Store conversation history in backend, send full context each time
- OR: Use session files that Claude CLI can read

### Scheduled Reminders

Using `node-cron`:
```javascript
// Daily 8am reminder
cron.schedule('0 8 * * *', async () => {
  await sendPushNotification({
    title: "Morning Check-in",
    body: "Ready to set your MIT for today?",
    data: { command: "start-day" }
  });
});

// Evening 8pm reminder
cron.schedule('0 20 * * *', async () => {
  await sendPushNotification({
    title: "Day Review",
    body: "How'd it go today?",
    data: { command: "end-day" }
  });
});
```

### Push Notifications (Firebase)

**Setup**:
1. Create Firebase project
2. Download service account key
3. Install `firebase-admin`
4. Store device tokens when app registers

**Send Logic**:
```javascript
await admin.messaging().send({
  token: userDeviceToken,
  notification: {
    title: "Coach",
    body: "Is this what you actually want?"
  },
  data: {
    type: "reminder",
    command: "check-day"
  }
});
```

### Deployment on Pi5

**Using PM2**:
```bash
npm install -g pm2
pm2 start src/server.js --name life-assistant-api
pm2 startup
pm2 save
```

**Nginx Reverse Proxy** (if needed):
```nginx
server {
  listen 80;
  server_name your-domain.com;

  location /api {
    proxy_pass http://localhost:3000;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection 'upgrade';
  }
}
```

**SSL with Let's Encrypt**:
```bash
sudo certbot --nginx -d your-domain.com
```

---

## Part 2: Mobile Interface

### Tech Stack (PWA - Recommended for MVP)
- **Framework**: Plain HTML/CSS/JS OR React (if you prefer)
- **UI**: Simple responsive CSS (mobile-first)
- **HTTP**: Fetch API
- **Storage**: LocalStorage
- **"Install"**: Add to Home Screen (iOS/Android)
- **Notifications**: Browser push (limited on iOS) OR just polling

### Alternative: React Native + Expo
- **Framework**: React Native + Expo
- **Navigation**: React Navigation
- **State**: Context API (simple)
- **HTTP**: Fetch API
- **Push**: Expo Notifications
- **UI**: React Native Paper
- **Storage**: AsyncStorage

**Recommendation**: Start with PWA. Simpler, faster, no app store. Upgrade to React Native later if needed.

### Project Structure
```
mobile/
├── App.js
├── app.json
├── src/
│   ├── screens/
│   │   ├── ChatScreen.js
│   │   ├── QuickActionsScreen.js
│   │   ├── SettingsScreen.js
│   │   └── ContextViewScreen.js
│   ├── components/
│   │   ├── ChatBubble.js
│   │   ├── QuickActionButton.js
│   │   ├── ReminderCard.js
│   │   └── InputBar.js
│   ├── services/
│   │   ├── api.js             # Backend API client
│   │   └── notifications.js   # Push notification handler
│   ├── store/
│   │   └── useStore.js        # Zustand store
│   ├── navigation/
│   │   └── AppNavigator.js
│   └── utils/
│       └── constants.js
└── package.json
```

### Key Screens

#### 1. Chat Screen (Main)
```
┌─────────────────────────┐
│  ← Coach        ⚙       │
├─────────────────────────┤
│                         │
│  [Coach]: Ready to      │
│  review your day?       │
│                         │
│         [You]: Yeah,    │
│         shipped the API │
│                         │
│  [Coach]: MIT done.     │
│  What's blocking you    │
│  tomorrow?              │
│                         │
├─────────────────────────┤
│ [____________] [Send]   │
└─────────────────────────┘
```

#### 2. Quick Actions Screen
```
┌─────────────────────────┐
│  Quick Actions          │
├─────────────────────────┤
│  ┌───────────────────┐ │
│  │  🌅 Start Day     │ │
│  └───────────────────┘ │
│  ┌───────────────────┐ │
│  │  ✅ Check-in      │ │
│  └───────────────────┘ │
│  ┌───────────────────┐ │
│  │  🌙 End Day       │ │
│  └───────────────────┘ │
│  ┌───────────────────┐ │
│  │  🤔 Decision Test │ │
│  └───────────────────┘ │
└─────────────────────────┘
```

#### 3. Settings Screen
```
┌─────────────────────────┐
│  Settings               │
├─────────────────────────┤
│  Reminders              │
│  ○ Morning (8:00 AM)    │
│  ○ Check-in (2:00 PM)   │
│  ○ Evening (8:00 PM)    │
│                         │
│  Server                 │
│  API URL: [________]    │
│  API Key: [********]    │
│                         │
│  Context Files          │
│  Last synced: 2 min ago │
│  [View CLAUDE.md]       │
│  [View NOW.md]          │
└─────────────────────────┘
```

### Core Features

**1. Chat Interface**
- Send message to backend
- Display conversation
- Auto-scroll to bottom
- Typing indicator
- Pull-to-refresh for context sync

**2. Quick Actions**
- Button triggers specific slash command
- Opens chat with command pre-loaded
- Visual feedback

**3. Push Notifications**
- Register device token on app start
- Handle notification tap → open relevant screen
- Badge count for unread messages

**4. Offline Support** (Optional)
- Cache last conversation
- Queue messages when offline
- Sync when back online

### Implementation Steps

#### Backend API (Week 1)

**Day 1-2: Core Setup**
1. Initialize Node.js project
2. Set up Express server
3. Configure Claude API integration
4. Implement file reading (CLAUDE.md, NOW.md)
5. Build `/api/chat` endpoint
6. Test with Postman

**Day 3: File Management**
7. Build `/api/context` endpoint
8. Build `/api/update-now` endpoint
9. Add validation and error handling

**Day 4: Reminders**
10. Set up node-cron
11. Build `/api/reminders` CRUD
12. Configure Firebase Admin
13. Implement push notification sending

**Day 5: Deployment**
14. Deploy to Pi5 with PM2
15. Set up nginx reverse proxy
16. Configure SSL
17. Test all endpoints from external network

#### Mobile App (Week 2)

**Day 1-2: Setup & Navigation**
1. Initialize React Native with Expo
2. Set up navigation structure
3. Create basic screen layouts
4. Configure Axios for API calls
5. Set up Zustand store

**Day 3: Chat Interface**
6. Build ChatScreen UI
7. Implement message sending
8. Display conversation history
9. Add pull-to-refresh

**Day 4: Quick Actions & Settings**
10. Build QuickActionsScreen
11. Wire up slash command buttons
12. Build SettingsScreen
13. Implement reminder configuration

**Day 5: Push Notifications**
14. Configure Expo Notifications
15. Register device token with backend
16. Handle notification tap
17. Test end-to-end flow

**Day 6-7: Polish & Testing**
18. UI refinements
19. Error handling
20. Loading states
21. End-to-end testing
22. Build for TestFlight/Internal Testing

---

## Environment Setup

### Backend .env
```
# Server
PORT=3000
NODE_ENV=production

# Claude API
ANTHROPIC_API_KEY=sk-ant-...

# Files
CLAUDE_MD_PATH=/home/user/claude_life_assistant/CLAUDE.md
NOW_MD_PATH=/home/user/claude_life_assistant/NOW.md

# Firebase
FIREBASE_SERVICE_ACCOUNT=/path/to/firebase-key.json

# Auth
API_SECRET_KEY=your-secret-key-here

# Git repo path
REPO_PATH=/home/user/claude_life_assistant
```

### Mobile app.json
```json
{
  "expo": {
    "name": "Life Coach",
    "slug": "life-coach",
    "version": "1.0.0",
    "extra": {
      "apiUrl": "https://your-pi5-domain.com/api",
      "apiKey": "your-api-key"
    }
  }
}
```

---

## ⚠️ CRITICAL RISKS (Claude CLI Approach)

This approach is a **hack** and comes with real risks:

1. **Not Officially Supported**: Claude Code CLI is not designed as a backend service
2. **May Break**: Anthropic could change CLI behavior, add rate limits, or block this usage
3. **Rate Limiting Risk**: Unclear if Max plan has usage limits for CLI automation
4. **Session Management**: CLI sessions are ephemeral, harder to maintain context
5. **Error Handling**: CLI output parsing can be brittle
6. **No SLA**: If it breaks, you're on your own

**Mitigation**:
- Build it anyway (free is free)
- Be ready to pivot to Claude API if needed
- Keep backend abstracted so switching is easy
- Monitor for any issues from Anthropic

**If Anthropic blocks this**: Fall back to manual Claude app usage or pay for API.

---

## Security Considerations

1. **API Authentication**: Simple API key for v1 (single user)
2. **HTTPS**: Use Let's Encrypt OR self-signed cert for testing
3. **Rate Limiting**: Add basic rate limiting to prevent abuse
4. **Input Validation**: Sanitize all inputs before passing to CLI
5. **Secrets Management**: API key in .env, never commit
6. **Firewall**: Only expose necessary ports on Pi5

---

## Cost Breakdown (REVISED)

- **Infrastructure**: $0 (using existing Pi5 + cloud server)
- **Claude Max Plan**: $200/mo (already paying)
- **Claude Code CLI**: Included in Max plan ✓
- **Firebase**: Free tier (up to 10k notifications/day) OR skip for MVP
- **Domain/SSL**: $12/year (optional, can use IP + self-signed for testing)
- **Apple Developer**: $0 (using PWA, no App Store needed)
- **Total Additional Cost**: **$0-12/year**

**Key Savings**: Using Claude CLI instead of API = **$600-1800/year saved**

---

## Success Metrics

**Week 1 Backend Complete When**:
- [ ] Chat endpoint responds with Claude-generated replies
- [ ] CLAUDE.md + NOW.md read correctly
- [ ] Push notification sent on schedule
- [ ] API accessible from phone's network

**Week 2 Mobile Complete When**:
- [ ] Can send message and get coach response
- [ ] Quick actions trigger correct prompts
- [ ] Reminders arrive on time
- [ ] App runs on physical device

---

## Future Enhancements (Post-MVP)

- Voice input/output
- Rich text formatting in chat
- Data visualization (habit tracking, MIT completion rate)
- Multiple coaches/personalities
- Shared accountability (invite accountability partner)
- Integration with calendar/task apps
- Conversation export

---

## Simplified MVP Roadmap (2-3 Days)

### Day 1: Minimal Backend
**Goal**: Get basic chat working via Claude CLI

1. Create `backend/` directory
2. Initialize Node.js project (`npm init -y`)
3. Install: `express`, `cors`, `dotenv`
4. Create single `server.js` with:
   - POST `/api/chat` endpoint
   - Wraps `claude` CLI via `child_process`
   - Returns response
5. Test with curl locally
6. Deploy to Pi5 with PM2

**Success**: `curl -X POST http://pi5:3000/api/chat -d '{"message":"hello"}'` returns Claude response

### Day 2: Simple Web UI
**Goal**: Mobile-friendly chat interface

1. Create `frontend/` directory
2. Single HTML file with:
   - Chat bubbles (flex column)
   - Input box at bottom
   - Fetch to backend `/api/chat`
   - Responsive CSS (mobile-first)
3. Serve via nginx or `python -m http.server`
4. Test on phone browser

**Success**: Open on phone, send message, get response

### Day 3: Quick Actions + Polish
**Goal**: Add slash commands and basic UX

1. Add buttons for `/start-day`, `/end-day`, `/check-day`
2. Add loading spinner
3. Add error handling
4. Store API URL in localStorage
5. Add simple auth (API key in header)
6. Add to home screen instructions

**Success**: Usable life coach in your pocket

### Optional: Reminders (Later)
- Add node-cron for scheduled notifications
- Use Firebase OR simple email/SMS OR skip for MVP

---

## What We're NOT Building (For MVP)

- ❌ Complex conversation state
- ❌ User accounts / multi-user
- ❌ Push notifications (use manual check-ins)
- ❌ Offline mode
- ❌ React Native app
- ❌ Database
- ❌ WebSockets
- ❌ File upload

Keep it stupid simple. Ship fast, iterate based on actual usage.

---

## Next Steps

**Option 1: Build MVP (Recommended)**
- Start with Day 1 backend
- Get something working in 2-3 days
- See if Claude CLI approach actually works
- Validate you'll actually use it

**Option 2: Test First**
- Try running `claude` from terminal in this repo
- See how it responds to messages
- Manually test slash commands
- Decide if wrapping it will work

**Which path?**
