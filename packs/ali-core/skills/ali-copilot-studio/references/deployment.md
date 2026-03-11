# ali-copilot-studio - Deployment Reference

This reference provides detailed deployment patterns for Microsoft CoPilot Studio across channels.

---

## Supported Channels

### Channel Comparison

| Channel | Integration Method | Authentication | Deployment Complexity |
|---------|-------------------|----------------|----------------------|
| **Teams** | Native app | Microsoft 365 SSO | Medium |
| **Web** | Embed code | Anonymous or SSO | Low |
| **Mobile** | Power Apps integration | Delegated auth | Medium |
| **Power Apps** | Component | App context | Low |
| **Dynamics 365** | Embedded widget | Dynamics auth | High |

---

## Teams Deployment

### Deployment Steps

**1. Package agent as Teams app**
- Navigate to CoPilot Studio > Publish
- Select "Microsoft Teams"
- Generate Teams app package (.zip)
- Download package

**2. Upload to Teams admin center**
- Navigate to Teams Admin Center
- Go to Teams apps > Manage apps
- Click "Upload"
- Select app package .zip file

**3. Configure app permissions**
- Set app availability (organization-wide, specific teams, or users)
- Configure data access permissions
- Set messaging policies
- Enable/disable app features (tabs, bots, connectors)

**4. Publish to organization or specific teams**

**Organization-wide:**
- Teams Admin Center > Teams apps > Setup policies
- Add app to global policy
- All users can install app

**Specific teams:**
- Navigate to team > Apps
- Click "Manage team" > Apps
- Add custom app
- Members can access immediately

**5. Enable in Teams client**
- Users open Teams
- Click Apps in left rail
- Search for app name
- Click "Add" to install

### Teams App Manifest

**Example manifest.json:**
```json
{
  "manifestVersion": "1.13",
  "id": "00000000-0000-0000-0000-000000000000",
  "version": "1.0.0",
  "packageName": "com.company.supportbot",
  "name": {
    "short": "IT Support Bot",
    "full": "IT Support & Helpdesk Bot"
  },
  "description": {
    "short": "Get IT help instantly",
    "full": "IT Support Bot helps you reset passwords, request VPN access, and get help with common IT issues."
  },
  "developer": {
    "name": "Company IT",
    "websiteUrl": "https://it.company.com",
    "privacyUrl": "https://it.company.com/privacy",
    "termsOfUseUrl": "https://it.company.com/terms"
  },
  "icons": {
    "color": "color-icon.png",
    "outline": "outline-icon.png"
  },
  "accentColor": "#0078D4",
  "bots": [
    {
      "botId": "00000000-0000-0000-0000-000000000000",
      "scopes": ["personal", "team", "groupchat"],
      "supportsFiles": true,
      "isNotificationOnly": false,
      "commandLists": [
        {
          "scopes": ["personal", "team"],
          "commands": [
            {
              "title": "Reset password",
              "description": "Reset your password"
            },
            {
              "title": "VPN access",
              "description": "Request VPN access"
            }
          ]
        }
      ]
    }
  ],
  "permissions": [
    "identity",
    "messageTeamMembers"
  ],
  "validDomains": [
    "company.com"
  ]
}
```

### Teams-Specific Features

**1. Adaptive Cards**
- Rich interactive messages
- Buttons, forms, images
- Submit actions back to agent

**Example:**
```json
{
  "type": "AdaptiveCard",
  "version": "1.3",
  "body": [
    {
      "type": "TextBlock",
      "text": "Ticket #12345 created",
      "weight": "bolder",
      "size": "large"
    },
    {
      "type": "TextBlock",
      "text": "Priority: High | Assigned to: John Doe"
    }
  ],
  "actions": [
    {
      "type": "Action.Submit",
      "title": "View Ticket",
      "data": {
        "action": "view_ticket",
        "ticket_id": "12345"
      }
    }
  ]
}
```

**2. Proactive messaging**
- Send notifications to users/channels
- Trigger from Power Automate
- Requires user consent

**3. Meeting extensions**
- Available during Teams meetings
- Pre-meeting, in-meeting, post-meeting
- Access meeting context

---

## Web Embed Deployment

### Basic Embed Code

```html
<iframe
  src="https://copilotstudio.microsoft.com/environments/{env}/bots/{bot}/webchat"
  style="width: 400px; height: 600px; border: none;"
  frameborder="0"
  allow="microphone; camera">
</iframe>
```

### Customization Options

**1. Color scheme and branding**
```html
<iframe
  src="https://copilotstudio.microsoft.com/environments/{env}/bots/{bot}/webchat?theme=dark&accent=%230078D4&logo=https://company.com/logo.png"
  style="width: 400px; height: 600px;">
</iframe>
```

**2. Welcome message**
```javascript
window.addEventListener('load', function() {
  const iframe = document.getElementById('copilot-iframe');
  iframe.contentWindow.postMessage({
    type: 'setWelcomeMessage',
    message: 'Hello! How can I help you today?'
  }, '*');
});
```

**3. Suggested actions**
```javascript
{
  type: 'setSuggestedActions',
  actions: [
    'Reset password',
    'VPN access',
    'Talk to human'
  ]
}
```

**4. File upload controls**
```html
<iframe
  src="...&enableFileUpload=true&maxFileSize=10485760"
  ...>
</iframe>
```

**5. Custom CSS injection**
```javascript
{
  type: 'injectCSS',
  css: `
    .chat-bubble {
      background-color: #f0f0f0;
      border-radius: 12px;
    }
    .user-message {
      background-color: #0078D4;
      color: white;
    }
  `
}
```

### Authentication in Web Embed

**Anonymous (no auth):**
```html
<iframe src="...&authMode=anonymous" ...></iframe>
```

**SSO (Microsoft 365):**
```html
<iframe src="...&authMode=sso&loginHint={user_email}" ...></iframe>
```

**Custom auth:**
```javascript
// Obtain token from your auth system
const token = await getCustomAuthToken();

// Pass to iframe
iframe.contentWindow.postMessage({
  type: 'setAuthToken',
  token: token
}, '*');
```

### Responsive Design

```css
/* Mobile */
@media (max-width: 768px) {
  #copilot-iframe {
    width: 100vw;
    height: 100vh;
    position: fixed;
    top: 0;
    left: 0;
  }
}

/* Desktop */
@media (min-width: 769px) {
  #copilot-iframe {
    width: 400px;
    height: 600px;
    position: fixed;
    bottom: 20px;
    right: 20px;
    box-shadow: 0 4px 12px rgba(0,0,0,0.15);
  }
}
```

---

## Mobile Deployment

### Power Apps Integration

**1. Add CoPilot component**
- Open Power Apps Studio
- Add "Copilot" component
- Configure agent endpoint

**2. Pass user context**
```javascript
CoPilot.Context = {
  userId: User().Email,
  department: User().Department,
  location: User().Office
}
```

**3. Handle responses**
```javascript
If(CoPilot.LastMessage.type = "ticket_created",
  Navigate(TicketScreen, ScreenTransition.Fade,
    {ticketId: CoPilot.LastMessage.ticketId})
)
```

### Native Mobile App (Teams Mobile)

- Agent available in Teams mobile app automatically
- Same functionality as desktop
- Optimized for touch interaction
- Support for voice input

---

## Power Apps Embedded

### Setup

**1. Add to canvas app**
- Insert > AI > Copilot component
- Select published agent
- Configure size and position

**2. Bind to app context**
```
Copilot.UserContext = {
  userId: User().Email,
  accountId: Gallery.Selected.AccountId,
  role: User().SecurityRole
}
```

**3. Respond to agent actions**
```
If(Copilot.SelectedAction = "view_record",
  Navigate(DetailScreen, Fade, {recordId: Copilot.ActionData.id})
)
```

---

## Dynamics 365 Embedded

### Deployment Steps

**1. Install solution**
- Navigate to Dynamics 365 > Solutions
- Import CoPilot solution package
- Configure environment variables

**2. Add widget to form**
- Open form designer
- Add CoPilot control
- Set properties (agent ID, context fields)

**3. Configure context binding**
```javascript
// Pass current record context to agent
{
  recordId: formContext.data.entity.getId(),
  recordType: formContext.data.entity.getEntityName(),
  userId: Xrm.Utility.getGlobalContext().userSettings.userId
}
```

**4. Handle agent actions**
```javascript
// Create case from agent interaction
function createCaseFromAgent(data) {
  Xrm.WebApi.createRecord("incident", {
    title: data.title,
    description: data.description,
    customerid: formContext.data.entity.getId()
  }).then(
    function success(result) {
      // Navigate to case
      Xrm.Navigation.openForm({
        entityName: "incident",
        entityId: result.id
      });
    }
  );
}
```

---

## Deployment Best Practices

### Staged Rollout

**Phase 1: Pilot (10 users)**
- Select diverse user group
- Collect detailed feedback
- Monitor usage patterns
- Fix critical issues

**Phase 2: Limited (100 users)**
- Expand to department or team
- Validate scalability
- Refine based on feedback
- Measure success metrics

**Phase 3: Broad (1000+ users)**
- Roll out to larger organization
- Monitor for issues at scale
- Provide training materials
- Establish support channel

**Phase 4: General Availability**
- Available to all users
- Continuous monitoring
- Iterative improvements

### Monitoring During Deployment

**Key metrics:**
- Active users (daily, weekly)
- Conversation completion rate
- Error rate
- Escalation to human rate
- User satisfaction (surveys)

**Alerts:**
- Error rate >5% → Engineering team
- Escalation rate >30% → Product team
- Response time >5 seconds → Infrastructure team

### Rollback Procedures

**Preparation:**
- Document rollback steps
- Maintain previous version
- Test rollback in dev environment

**Execution:**
1. Disable new agent version
2. Re-enable previous version
3. Notify users of temporary disruption
4. Investigate issue
5. Fix and redeploy

---

## Cross-Channel Considerations

### Consistent Experience

**Challenge**: Same agent across Teams, web, mobile

**Solutions:**
- Use responsive design patterns
- Avoid channel-specific features in core flows
- Test thoroughly on all channels
- Provide fallbacks for unsupported features

### Channel-Specific Features

| Feature | Teams | Web | Mobile |
|---------|-------|-----|--------|
| Adaptive cards | ✅ | ❌ | ✅ |
| File upload | ✅ | ⚠️ (config) | ✅ |
| Voice input | ✅ | ⚠️ (browser) | ✅ |
| Proactive messages | ✅ | ❌ | ✅ |
| Rich formatting | ✅ | ⚠️ (limited) | ✅ |

### Testing Matrix

| Test Case | Teams | Web | Mobile | Power Apps | Dynamics |
|-----------|-------|-----|--------|-----------|----------|
| Basic conversation | ✓ | ✓ | ✓ | ✓ | ✓ |
| File upload | ✓ | ✓ | ✓ | - | - |
| Authentication | ✓ | ✓ | ✓ | ✓ | ✓ |
| Adaptive cards | ✓ | - | ✓ | - | - |
| Context passing | ✓ | ✓ | ✓ | ✓ | ✓ |

---

## Troubleshooting

### Agent Not Appearing in Teams

**Solutions**:
- Verify app is published in Teams Admin Center
- Check user has permission to install app
- Ensure app is enabled in organization setup policies
- Clear Teams cache and restart

### Web Embed Not Loading

**Solutions**:
- Check iframe src URL is correct
- Verify HTTPS (not HTTP)
- Check CORS settings allow embedding
- Inspect browser console for errors
- Test in incognito mode (eliminate browser extensions)

### Mobile Performance Issues

**Solutions**:
- Reduce image sizes in responses
- Minimize large data transfers
- Use pagination for long lists
- Optimize API response times
- Test on low-bandwidth connections
