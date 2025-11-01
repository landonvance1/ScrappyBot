# **Family Photo/Video Backup Bot - Design Document**

## **Executive Summary**

An SMS-based bot that automatically backs up photos and videos of your child to Google Cloud Storage. The bot handles the common problem of losing precious family moments that live only in text message threads.

**Key Features:**

* Conversational SMS interface for zero-friction backup
* Group photo sharing with private 1-on-1 confirmation flows
* One photo/video at a time per user to avoid confusion
* User confirmation before upload (always includes the media in confirmation)
* Organized storage by date
* Handles photo bursts gracefully with queueing
* Video support with thumbnail-based confirmation
* Secure whitelist of authorized users

---

## **Problem Statement**

Parents frequently share photos and videos of their children via SMS/MMS but fail to properly back them up. These memories remain trapped in message threads, unorganized, and at risk of being lost if phones are upgraded, messages are deleted, or storage fails.

**Current pain points:**

* Photos sent via text don't automatically backup to organized storage
* High friction in existing solutions (requires opening separate apps, manual uploads)
* Risk of losing irreplaceable memories

---

## **Solution Overview**

A conversational SMS bot that:

1. Receives photos/videos sent to a group MMS thread (parents + bot)
2. Responds in private 1-on-1 threads with each parent
3. **Always sends the photo/video back** in the confirmation message
4. Processes items one at a time per user
5. Requests confirmation before saving (with @scrappy prefix)
6. Uploads to Google Cloud Storage in date-organized folder structure
7. Handles timeouts gracefully (24h reminder, 48h cleanup)
8. Restricts access to authorized users only

**Core Principle:** Make backup as frictionless as possible while maintaining user control over what gets saved.

---

## **High-Level Architecture Decisions**

### **Key Design Decisions**

| Decision | Choice | Rationale |
|----------|--------|-----------|
| **Programming Language** | C# / .NET | Developer preference, excellent ecosystem, great Twilio/GCS SDKs available |
| **Messaging Platform** | SMS/MMS via Twilio | Ubiquitous, no new apps needed, works on any phone |
| **Photo Sharing** | Group thread (You + Wife + Bot) | Share photos once, both parents see them |
| **Confirmation Flow** | Private 1-on-1 threads | Keeps group chat clean |
| **Confirmation Always Includes Media** | Yes, always | User sees exactly what they're confirming, even for single photos |
| **Intent Signal** | @scrappy prefix required | Prevents bot from responding to regular conversation |
| **Organization** | Date-based only | Simple, intuitive, matches natural browsing patterns |
| **State Management** | Redis | Fast, ephemeral state perfect for conversation flow |
| **Temp Storage** | Local filesystem | Photos stored locally during confirmation process |
| **Cloud Storage** | Google Cloud Storage | Cost-effective, reliable, good C# SDK |
| **Authorization** | Phone number whitelist | Simple, effective security for private family bot |
| **Video Handling** | MMS-compressed, thumbnail confirmation | Accepts quality trade-off for convenience |

## **User Experience Flow**

### **Happy Path: Single Photo**

```
[Group Chat: You, Wife, Scrappy Bot]
You → Sends photo to group
     ↓
[1-on-1 Chat: You ↔ Scrappy Bot]
Bot → [Sends the photo back]
      "Should I save this photo? 
       Reply: @scrappy yes
           or: @scrappy no"
     ↓
You → "@scrappy yes"
     ↓
Bot → "✓ Saved to 2025/10/"
```

**Key point:** Bot ALWAYS sends the photo back in confirmation, not just for multiple photos.

### **Happy Path: Video**

```
[Group Chat: You, Wife, Scrappy Bot]
You → Sends video to group
     ↓
[1-on-1 Chat: You ↔ Scrappy Bot]
Bot → [Sends thumbnail of middle frame] 
      "📹 Video (25s, 4.2MB)
       Should I save this video?
         
       Reply: @scrappy yes
           or: @scrappy no"
     ↓
You → "@scrappy yes"
     ↓
Bot → "✓ Saved to 2025/10/"
```

### **Multiple Photos Scenario**

```
[Group Chat: You, Wife, Scrappy Bot]
You → Sends 5 photos in quick succession
     ↓
[1-on-1 Chat: You ↔ Scrappy Bot]
Bot → [Processes photo 1, queues 2-5]
      [Sends photo 1 back]
      "Should I save this photo?
       Reply: @scrappy yes or @scrappy no"
     ↓
You → "@scrappy yes"
     ↓
Bot → "✓ Saved to 2025/10/"
      [Auto-processes photo 2]
      [Sends photo 2 back]
      "Should I save this photo?..."
     ↓
(continues until queue empty)
```

### **Skip Photo**

```
[1-on-1 Chat: You ↔ Scrappy Bot]
Bot → [Sends photo back]
      "Should I save this photo?
       Reply: @scrappy yes or @scrappy no"
     ↓
You → "@scrappy no"
     ↓
Bot → "Got it, skipping this photo! 👍"
      [Processes next in queue]
```

### **Forgot @scrappy Prefix**

```
[1-on-1 Chat: You ↔ Scrappy Bot]
Bot → [Sends photo back]
      "Should I save this photo?
       Reply: @scrappy yes or @scrappy no"
     ↓
[Group Chat: You, Wife, Scrappy Bot]
You → "yes definitely!" (talking to wife)
     ↓
Bot → (ignores - no @scrappy prefix)
```

### **Timeout Scenario**

```
[Group Chat: You, Wife, Scrappy Bot]
You → Sends 3 photos
     ↓
[1-on-1 Chat: You ↔ Scrappy Bot]
Bot → [Sends photo 1 back]
      "Should I save this photo?..."
     ↓
You → [No response for 24 hours]
     ↓
Bot → "Hey! You have 3 photo(s) waiting. Reply to continue, 
       or I'll clear them in 24 hours."
     ↓
You → [Still no response for another 24 hours]
     ↓
Bot → "I haven't heard from you in a while, so I've cleared 
       the pending photos. Send new photos anytime!"
      [Queue dumped, files cleaned up]
```

---

## **System Architecture**

### **High-Level Architecture**

```
┌─────────────────────────────────────────────────────────────┐
│                         Twilio                              │
│                  (SMS/MMS Gateway)                          │
│  - Receives photos/videos via MMS                           │
│  - Sends confirmations and responses to 1-on-1 threads      │
└────────────────────────────┬────────────────────────────────┘
                             │ HTTPS Webhook
                             ▼
┌─────────────────────────────────────────────────────────────┐
│                   ASP.NET Core Web API                      │
│                   (C# Application)                          │
│                                                             │
│  Webhook Endpoints:                                         │
│  • POST /webhook/sms - Main entry point                     │
│  • GET /thumbs/{id} - Serve thumbnails for MMS              │
│  • GET /health - Health check endpoint                      │
│                                                             │
│  Core Components:                                           │
│  • Authorization Filter (whitelist check)                   │
│  • Message Router                                           │
│  • @scrappy Prefix Parser                                   │
│  • Photo Handler (download, store)                          │
│  • Video Handler (download, extract thumbnail)              │
│  • Conversation Engine (state machine)                      │
│  • GCS Uploader                                             │
│                                                             │
│  Background Services:                                       │
│  • Timeout Monitor (IHostedService)                         │
│  • File Cleanup Service                                     │
└────────────────────────────┬────────────────────────────────┘
                             │
                ┌────────────┴────────────┐
                │                         │
                ▼                         ▼
    ┌───────────────────┐    ┌──────────────────────┐
    │      Redis        │    │   Local Storage      │
    │  (State & Queue)  │    │   /tmp/photos        │
    │                   │    │                      │
    │ • User state      │    │ • Pending photos     │
    │ • Photo queue     │    │ • Pending videos     │
    │ • Current photo   │    │ • Thumbnails         │
    │ • Last active     │    │                      │
    └───────────────────┘    └──────────────────────┘

                             ▼
                ┌──────────────────────────┐
                │  Google Cloud Storage    │
                │                          │
                │  Organized by:           │
                │  • Date (year/month)     │
                │  • Metadata              │
                └──────────────────────────┘
```

### **Component Responsibilities**

**Twilio:**

* SMS/MMS gateway
* Receives media from group thread
* Delivers bot messages to individual 1-on-1 threads
* Handles carrier compatibility

**ASP.NET Core Web API:**

* Receives Twilio webhooks
* **Validates sender against authorized users whitelist**
* Parses @scrappy prefix from messages
* Routes messages to appropriate handlers
* Manages conversation state machine
* Orchestrates uploads to GCS
* Serves thumbnails for MMS delivery
* Background services for timeout monitoring

**Redis (via StackExchange.Redis):**

* Stores ephemeral user state
* Manages per-user photo queues
* Tracks conversation position
* Auto-expires old data (72h TTL)

**Local Storage (/tmp/photos or /app/photos):**

* Temporary storage for pending media
* Holds photos/videos during confirmation flow
* Stores generated thumbnails
* Cleaned up after upload or timeout

**Background Services (IHostedService):**

* Runs continuously or on schedule
* Monitors inactive users
* Sends 24h reminders
* Dumps queues after 48h
* Cleans orphaned temp files

**Google Cloud Storage:**

* Long-term storage for confirmed media
* Date-organized folder structure

---

## **Critical Workflows**

### **Workflow 1: Photo Received and Processed**

```
1. User sends photo to GROUP (You, Wife, Bot)
   ↓
2. Twilio webhook → POST /webhook/sms
   ↓
3. AUTHORIZATION CHECK
   - Is sender in AUTHORIZED_USERS?
   - If NO: Silently ignore, log, return 200
   - If YES: Continue
   ↓
4. Download photo from Twilio
   - Save to /tmp/photos/user_{phone}/photo_{id}.jpg
   ↓
5. Add photo_id to user's Redis queue
   ↓
6. Check user's current state
   - If IDLE: Start processing (step 7)
   - If BUSY: Just queue, return (will process later)
   ↓
7. Pop first photo from queue
   - Set as current photo in Redis
   - Change state to AWAITING_CONFIRM
   ↓
8. Send MMS to 1-ON-1 THREAD (not group!)
   - To: sender's individual phone number
   - Media: The photo they just sent (re-use Twilio URL)
   - Body: "Should I save this photo?
            
            Reply: @scrappy yes
                or: @scrappy no"
   ↓
9. Wait for user response...
   ↓
10. User sends text: "@scrappy yes"
    ↓
11. Parse @scrappy prefix
    - If no prefix: Send reminder, ignore
    - If prefix + "yes": Continue
    - If prefix + "no": Delete file, process next
    ↓
12. Change state to UPLOADING
    ↓
13. Upload to GCS
    - /2025/10/photo_xyz.jpg (date folder)
    ↓
14. Delete local file
    ↓
15. Send success SMS to 1-on-1
    - Body: "✓ Saved to 2025/10/"
    ↓
16. Change state to IDLE
    ↓
17. Check queue - any more photos?
    - If YES: Go to step 7
    - If NO: Done, stay IDLE
```

### **Workflow 2: Video Received and Processed**

```
1. User sends video to GROUP
   ↓
2. Twilio webhook → POST /webhook/sms
   ↓
3. AUTHORIZATION CHECK (same as photo)
   ↓
4. Download video from Twilio
   - Save to /tmp/photos/user_{phone}/photo_{id}.mp4
   ↓
5. Extract middle frame as thumbnail
   - Use FFmpeg (via Process or FFMediaToolkit)
   - Get video duration
   - Extract frame at duration/2
   - Save to /tmp/photos/user_{phone}/photo_{id}_thumb.jpg
   ↓
6. Add video_id to user's Redis queue
   ↓
7. If user IDLE, start processing:
   - Pop video from queue
   - Set as current
   - Change state to AWAITING_CONFIRM
   ↓
8. Serve thumbnail via /thumbs/{id} endpoint
   - Returns thumbnail file
   ↓
9. Send MMS to 1-ON-1 THREAD
   - To: sender's individual number
   - Media: https://your-domain.com/thumbs/{id}
   - Body: "📹 Video (25s, 4.2MB)
            Should I save this video?
              
            Reply: @scrappy yes
                or: @scrappy no"
   ↓
10. User replies "@scrappy yes"
    ↓
11. Upload to GCS:
    - Video file to date folder
    - Thumbnail to date folder
    - Metadata JSON with duration and size
    ↓
12. Delete local files
    ↓
13. Send success SMS: "✓ Saved to 2025/10/"
    ↓
14. Change state to IDLE, process next in queue
```

### **Workflow 3: Multiple Photos (Queue Processing)**

User sends 5 photos rapidly (within 30 seconds):

```
Time 0s: Photo 1 arrives
  - Authorization: ✓
  - Download, save locally
  - Add to queue: [Photo1]
  - User state: IDLE
  - Pop Photo1, start processing
  - Send confirmation to 1-on-1 (with Photo1)
  - State: AWAITING_CONFIRM

Time 5s: Photos 2, 3 arrive
  - Authorization: ✓ (both)
  - Download, save locally
  - Add to queue: [Photo1 (processing), Photo2, Photo3]
  - User state: AWAITING_CONFIRM (busy)
  - Just queue, don't process yet

Time 10s: Photos 4, 5 arrive
  - Authorization: ✓ (both)
  - Download, save locally
  - Queue: [Photo1 (processing), Photo2, Photo3, Photo4, Photo5]
  - User still busy, just queue

Time 15s: User replies "@scrappy yes" to Photo1
  - Upload Photo1
  - Send success message: "✓ Saved to 2025/10/"
  - State: IDLE
  - Queue not empty, so immediately:
    - Pop Photo2
    - Send confirmation (with Photo2)
    - State: AWAITING_CONFIRM
  - Queue: [Photo3, Photo4, Photo5]

Repeat until queue empty
```

### **Workflow 4: Authorization Enforcement**

ANY incoming message:

```
1. Extract from_number from Twilio webhook
   ↓
2. Check whitelist:
   if (AUTHORIZED_USERS.ContainsKey(from_number))
   {
       // Authorized - continue processing
   }
   else
   {
       // Unauthorized
       Log.Warning("Unauthorized attempt from {Phone}", from_number);
       return Ok(); // Return 200 to Twilio (don't retry)
       // Do NOT send response to sender
       // Do NOT process message
   }
```

### **Workflow 5: @scrappy Prefix Parsing**

User sends any text message:

```
1. Extract message body
   ↓
2. Check for @scrappy prefix:
     
   var prefixes = new[] { 
       "@scrappy", "scrappy", "@scrap", "scrap", 
       "@s", "s:", "hey scrappy" 
   };
     
   bool isForBot = false;
   string command = "";
     
   foreach (var prefix in prefixes)
   {
       if (body.StartsWith(prefix, StringComparison.OrdinalIgnoreCase))
       {
           isForBot = true;
           command = body.Substring(prefix.Length).Trim();
           command = command.TrimStart(':').Trim();
           break;
       }
   }
   ↓
3. If NOT for bot:
   - Check if user has pending action
   - Otherwise: Ignore silently
   ↓
4. If FOR bot:
   - Get user's current state
   - Route to appropriate handler:
     - AWAITING_CONFIRM → ParseConfirmation(command)
     - IDLE → SendHelp()
```

### **Workflow 6: Timeout Handling**

Background Service runs every hour:

```
1. Get all users from Redis
   foreach (var user in AUTHORIZED_USERS)
   {
       var state = GetUserState(user.PhoneNumber);
       var lastActive = state.LastActive;
       var hoursInactive = (DateTime.Now - lastActive).TotalHours;
         
       if (hoursInactive >= 48)
       {
           // DUMP QUEUE
           DumpUserQueue(user.PhoneNumber);
           SendSMS(user.PhoneNumber, 
               "I haven't heard from you in a while, " +
               "so I've cleared the pending photos.");
       }
       else if (hoursInactive >= 24 && !ReminderSent(user.PhoneNumber))
       {
           // SEND REMINDER
           var pendingCount = GetQueueLength(user.PhoneNumber) + 
                             (HasCurrentPhoto(user.PhoneNumber) ? 1 : 0);
             
           if (pendingCount > 0)
           {
               SendSMS(user.PhoneNumber,
                   $"Hey! You have {pendingCount} photo(s) waiting. " +
                   "Reply to continue, or I'll clear them in 24 hours.");
                 
               SetReminderSent(user.PhoneNumber);
           }
       }
   }

2. Clean orphaned files
   - List all files in /tmp/photos
   - Check if referenced in any Redis queue
   - If not referenced OR older than 1 week: Delete
```

---

## **State Machine**

### **States**

```
IDLE
  ↓ (photo/video received)
AWAITING_CONFIRM
  ↓ (user replies "@scrappy yes")
UPLOADING
  ↓ (upload complete)
IDLE (process next in queue or stay idle)
```

### **State Transitions**

```csharp
public enum ConversationState
{
    Idle,
    AwaitingConfirm,
    Uploading
}

// Transitions:
// 1. IDLE → AWAITING_CONFIRM
//    Trigger: New media received, user has no pending conversation
//    Action: Download, queue, pop first, send confirmation with media

// 2. AWAITING_CONFIRM → UPLOADING
//    Trigger: User replies "@scrappy yes"
//    Action: Begin upload

// 3. AWAITING_CONFIRM → IDLE
//    Trigger: User replies "@scrappy no"
//    Action: Delete file, process next in queue

// 4. UPLOADING → IDLE
//    Trigger: Upload completes
//    Action: Delete local file, send success, process next

// 5. Any State → IDLE
//    Trigger: 48 hours inactive
//    Action: Dump queue, clean files, notify user
```

---

## **Data Model**

### **Redis Keys**

```
// User State
user:{phone_number}:state
{
    "state": "Idle" | "AwaitingConfirm" | "Uploading",
    "lastActive": "2025-10-26T14:30:00Z"
}
TTL: 72 hours

// User Queue (LIST)
user:{phone_number}:queue
["photo_id1", "photo_id2", "photo_id3"]
TTL: 72 hours

// Current Media
user:{phone_number}:current
{
    "photoId": "photo_20251026_140032_abc123",
    "type": "Photo" | "Video",
    "localPath": "/tmp/photos/user_+1234567890/photo_xyz.jpg",
    "thumbnailPath": "/tmp/photos/..._thumb.jpg",  // videos only
    "receivedAt": "2025-10-26T14:00:00Z",
    "twilioUrl": "https://api.twilio.com/...",
    "sizeBytes": 2458392,
    "durationSeconds": 25  // videos only
}
TTL: 72 hours

// Photo/Video Metadata
photo:{photo_id}
{
    "type": "Photo" | "Video",
    "localPath": "/tmp/photos/...",
    "thumbnailPath": "/tmp/photos/...",  // videos only
    "receivedAt": "2025-10-26T14:00:00Z",
    "twilioUrl": "https://...",
    "sizeBytes": 2458392,
    "durationSeconds": 25  // videos only
}
TTL: 72 hours

// Reminder Flag
user:{phone_number}:reminder_sent
Value: "true"
TTL: 48 hours
```

### **Configuration: Authorized Users**

```json
// appsettings.json or environment variables
{
  "AuthorizedUsers": {
    "+12345678901": "Dad",
    "+19876543210": "Mom"
  }
}

// Used for:
// 1. Whitelist checking
// 2. Friendly names in logs/messages
// 3. Audit trail
```

### **Local File Structure**

```
/tmp/photos/  (or /app/photos in container)
├── user_+12345678901/
│   ├── photo_20251026_140032_abc123.jpg          # photo
│   ├── photo_20251026_140145_def456.mp4          # video
│   ├── photo_20251026_140145_def456_thumb.jpg    # video thumbnail
│   └── photo_20251026_141203_ghi789.jpg          # photo
└── user_+19876543210/
    └── photo_20251026_135500_xyz999.jpg
```

### **GCS Folder Structure**

```
your-bucket-name/
├── 2025/
│   └── 10/
│       ├── photo_20251026_140032_abc123.jpg
│       ├── photo_20251026_140145_def456.mp4
│       ├── photo_20251026_140145_def456_thumb.jpg
│       └── photo_20251026_141203_ghi789.jpg

```

---

## **Technology Stack**

### **Core Application**

* **Language:** C# / .NET 8+
* **Web Framework:** ASP.NET Core Web API

### **State Management**

* **Database:** Redis 7+
* **Library:** StackExchange.Redis

### **External Services**

* **SMS/MMS:** Twilio
  * NuGet: `Twilio`
  * Webhook handling, MMS send/receive
* **Cloud Storage:** Google Cloud Storage
  * NuGet: `Google.Cloud.Storage.V1`
  * Reliable, cost-effective
* **Video Processing:** FFmpeg
  * Call via `System.Diagnostics.Process`
  * Or use `FFMediaToolkit` NuGet package

### **Infrastructure**

* **Hosting:** Docker container on VPS
  * Single container with ASP.NET Core app
  * Separate Redis container
  * Ubuntu 24.04 host
* **Background Services:** IHostedService
  * Built into ASP.NET Core
  * Timeout monitoring
  * File cleanup

---

## **Deployment Architecture**

```
┌───────────────────────────────────────────────────────┐
│                    VPS Server                         │
│                                                       │
│  ┌────────────────────────────────────────────────┐   │
│  │           Docker Compose                       │   │
│  │                                                │   │
│  │  ┌──────────────┐      ┌──────────────┐        │   │
│  │  │  ASP.NET     │      │    Redis     │        │   │
│  │  │   Core       │◄────►│   Container  │        │   │
│  │  │  Container   │      │              │        │   │
│  │  │ Port 8080    │      │ Port 6379    │        │   │
│  │  └──────────────┘      └──────────────┘        │   │
│  │         ▲                                      │   │
│  │         │                                      │   │
│  │  ┌──────┴───────┐                              │   │
│  │  │    Nginx     │                              │   │
│  │  │ (reverse     │                              │   │
│  │  │  proxy)      │                              │   │
│  │  │ Port 80/443  │                              │   │
│  │  └──────────────┘                              │   │
│  └────────────────────────────────────────────────┘   │
│                                                       │
│  Volume Mounts:                                       │
│  • /app/photos → Host filesystem                      │
│  • Redis data → Named volume                          │
└───────────────────────────────────────────────────────┘
           ▲                            │
           │                            │
    HTTPS  │                            │ HTTPS
  (webhook)│                            │ (upload)
           │                            ▼
      ┌─────────┐              ┌──────────────┐
      │ Twilio  │              │    Google    │
      │         │              │    Cloud     │
      │ SMS/MMS │              │   Storage    │
      └─────────┘              └──────────────┘
```

---

### **Twilio Signature Validation**

```csharp
using Twilio.Security;

[HttpPost("webhook/sms")]
public IActionResult ReceiveMessage()
{
    // Validate Twilio signature
    var validator = new RequestValidator(_twilioAuthToken);
    var signature = Request.Headers["X-Twilio-Signature"];
    var url = $"{Request.Scheme}://{Request.Host}{Request.Path}{Request.QueryString}";
    var parameters = Request.Form.ToDictionary(
        x => x.Key, 
        x => x.Value.ToString());
      
    if (!validator.Validate(url, parameters, signature))
    {
        _logger.LogError("Invalid Twilio signature");
        return Forbid();
    }
      
    // Process message...
}
```

### **Data Privacy**

* **Local storage:** Not web-accessible, auto-cleanup
* **GCS bucket:** Private, IAM-controlled
* **Redis:** Internal network only
* **All communication:** HTTPS/TLS

---

## **Message Parsing: @scrappy Prefix**

### **The Problem**

Bot cannot distinguish message context:

* Group chat to spouse: "yes, definitely!"
* 1-on-1 to bot: "yes"

Both arrive identically from Twilio.

### **The Solution**

All bot commands require @scrappy prefix.

---

## **Cost Estimate**

### **Monthly Operating Costs (2 users, moderate usage)**

| Item | Quantity | Unit Cost | Monthly Cost |
|------|----------|-----------|--------------|
| **Infrastructure** | | | |
| VPS Hosting | 1 server (2GB RAM) | $10/month | $10.00 |
| **Twilio** | | | |
| Phone Number | 1 number | $1.50/month | $1.50 |
| Incoming MMS | 630 messages | $0.0075 | $4.73 |
| Outgoing MMS (confirmations) | 630 messages | $0.0079 | $4.98 |
| Outgoing SMS (other) | 180 messages | $0.0079 | $1.42 |
| **Google Cloud Storage** | | | |
| Storage | 2 GB | $0.02/GB | $0.04 |
| Egress | 500 MB | $0.12/GB | $0.06 |
| | | **Total:** | **$22.73/month** |

**Annual: ~$273/year**

---

## **Key Takeaways**

### **What Makes This Design Work**

1. **Group + 1-on-1 Split:** Share photos in group, confirm privately
2. **Always Show Media:** User always sees what they're confirming
3. **@scrappy Prefix:** Prevents accidental bot triggers
4. **Per-User Queues:** Independent processing, no interference
5. **Authorization Whitelist:** Simple, effective security
6. **Timeouts:** Graceful cleanup prevents stale state
7. **Date-Only Organization:** Simple, intuitive, matches natural browsing
8. **C# + .NET:** Great SDKs, performance, developer productivity

### **Critical Success Factors**

* **Forgiving @scrappy parsing** - Accept typos/variations
* **Clear prompts** - Always show exact format to use
* **Helpful reminders** - Detect when user forgets prefix
* **Visual confirmation** - Always send media back
* **One-at-a-time processing** - Prevents confusion
* **Reliable cleanup** - Timeout handling essential
* **Simple organization** - Date-based folders, no categories to remember

### **Future Considerations**

* Web gallery for browsing
* AI auto-categorization (optional tagging)
* Send random photos to users ("memory of the day")
* Multiple children support (separate bots or folder structure)
* Print integration

---

**End of Design Document**