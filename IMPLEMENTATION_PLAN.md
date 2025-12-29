# Life Assistant Mobile App - Implementation Plan

## Overview
Build a mobile app with clean UI that acts as your personal coach, powered by Claude API, reading from CLAUDE.md/NOW.md files stored on Pi5/server.

---

## Architecture

### System Components

```
┌─────────────────────────────────────────┐
│         Mobile App (React Native)       │
│  ┌─────────────────────────────────┐   │
│  │  Chat Interface                  │   │
│  │  Quick Actions (/start-day, etc)│   │
│  │  Settings & Reminders            │   │
│  │  Push Notification Handler       │   │
│  └─────────────────────────────────┘   │
└──────────────┬──────────────────────────┘
               │ HTTPS/WSS
               │
┌──────────────▼──────────────────────────┐
│       Backend API (Node.js/Express)     │
│  ┌─────────────────────────────────┐   │
│  │  /api/chat                       │   │
│  │  /api/update-now                 │   │
│  │  /api/get-context                │   │
│  │  /api/reminders                  │   │
│  │  WebSocket for real-time updates│   │
│  └─────────────────────────────────┘   │
└──────────────┬──────────────────────────┘
               │
        ┌──────┴──────┐
        │             │
┌───────▼──────┐  ┌──▼────────────┐
│ Claude API   │  │ File System   │
│ (Anthropic)  │  │ CLAUDE.md     │
│              │  │ NOW.md        │
└──────────────┘  └───────────────┘
```

---

## Part 1: Backend API

### Tech Stack
- **Runtime**: Node.js 20+
- **Framework**: Express.js
- **Claude Integration**: @anthropic-ai/sdk
- **File Operations**: fs/promises
- **Scheduler**: node-cron
- **Push Notifications**: Firebase Admin SDK
- **Auth**: API Key + JWT
- **WebSocket**: Socket.io (optional for real-time)

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

### Claude Integration Strategy

**Context Building**:
```javascript
// On every chat request:
const systemPrompt = `
${fs.readFileSync('CLAUDE.md', 'utf-8')}

${fs.readFileSync('NOW.md', 'utf-8')}

You are the agent described in CLAUDE.md.
Current state is in NOW.md.
Respond according to your rules and personality.
`;

const response = await anthropic.messages.create({
  model: 'claude-sonnet-4-5-20250929',
  max_tokens: 1024,
  system: systemPrompt,
  messages: conversationHistory
});
```

**Conversation Memory**:
- Store last 10 messages in Redis or in-memory (simple Map)
- Or stateless: send full CLAUDE.md + NOW.md each time (works with Claude's long context)

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

## Part 2: Mobile App (React Native)

### Tech Stack
- **Framework**: React Native + Expo
- **Navigation**: React Navigation
- **State**: Zustand (lightweight) or Context API
- **HTTP**: Axios
- **Push**: Expo Notifications
- **UI**: React Native Paper or NativeBase
- **Storage**: AsyncStorage

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

## Security Considerations

1. **API Authentication**: Simple API key for v1, JWT for multi-user later
2. **HTTPS Only**: Use SSL cert on Pi5
3. **Rate Limiting**: Prevent API abuse
4. **Input Validation**: Sanitize all inputs
5. **Secrets Management**: Never commit API keys
6. **Firebase Rules**: Restrict who can send push notifications

---

## Cost Breakdown

- **Infrastructure**: $0 (using existing Pi5 + cloud server)
- **Claude API**: Included in Max plan (monitor usage)
- **Firebase**: Free tier (up to 10k notifications/day)
- **Domain/SSL**: $12/year (optional, can use IP)
- **Total**: ~$0-12/year

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

## Next Steps

1. **Review this plan** - Any changes needed?
2. **Start backend scaffolding** - Set up project structure
3. **Deploy to Pi5** - Get it running
4. **Build mobile app** - UI + API integration
5. **Test end-to-end** - Full flow with real device

Ready to start scaffolding the backend?
