# Zoom Websocket messages details for lock, unlock and update:

Here's the detailed breakdown for **each message type including request payloads and response structures** for the frontend team to implement. This includes **locking, unlocking, and updating**.

---

## **1. Lock**

### **Request Payload (Sent from Frontend)**

When an editor requests to acquire a lock for a specific timestamp:
```json
{
    "action" : "lock",
    "data": {
        "timestamp": "12345"  // The TS (Timestamp) they want to lock
    }
}
```

---

### **Responses**

#### **Success: `lock-acquired`**
Broadcasted to all editors when the lock is successfully acquired.

**Response Structure:**
```json
{
    "type": "lock-acquired",
    "data": {
        "timestamp": "12345",   // The TS that has been locked
        "editor": "sessionIdA" // The ID of the editor/session who acquired the lock
    }
}
```

#### **Failure: `lock-failure`**
Sent to the requesting editor if the lock cannot be acquired (e.g., it’s already held by another editor).

**Response Structure:**
```json
{
    "type": "lock-failure",
    "data": {
        "timestamp": "12345",  // The TS the editor attempted to lock
        "error": "Lock not available or already held by another editor." // Reason for failure
    }
}
```

---

## **2. Unlock**

### **Request Payload (Sent from Frontend)**

When an editor requests to release the lock for a specific timestamp:
```json
{
    "action": "unlock",
    "data": {
        "timestamp": "12345"  // The TS (Timestamp) they want to release
    }
}
```

---

### **Responses**

#### **Success: `lock-released`**
Broadcasted to all editors when the lock is successfully released.

**Response Structure:**
```json
{
    "type": "lock-released",
    "data": {
        "timestamp": "12345",   // The TS that has been unlocked
        "editor": "sessionIdA" // The ID of the editor/session who released the lock
    }
}
```

This lets all editors know the timestamp is now available for editing.

---

## **3. Update**

### **Request Payload (Sent from Frontend)**

When an editor submits changes to the transcript for some timestamps:
```json
{
    "action": "update",
    "data": {
        "changes": [
            {
                "timestamp": "12345",   // The TS of the transcript being updated
                "text": "New text content" // The updated text for the transcript
            },
            {
                "timestamp": "12346",
                "text": "Another updated text"
            }
        ]
    }
}
```

- `changes` is an array of updates where each update contains:
  - The `timestamp` of the transcript being updated.
  - The new `text` for that timestamp.

---

### **Responses**

#### **Success: `update`**
Broadcasted to all editors when the update is successfully applied to the transcript.

**Response Structure:**
```json
{
    "type": "update",
    "data": [
        {
            "timestamp": "12345",        // The TS that was updated
            "text": "New text content",  // The updated text
            "editor": "sessionIdA"       // The ID of the editor/session who made the update
        },
        {
            "timestamp": "12346",
            "text": "Another updated text",
            "editor": "sessionIdA"
        }
    ]
}
```

This informs all editors to reflect the latest updates in their UI.

#### **Failure: `update-failure`**
Sent to the editor who submitted the update if:
- The lock for a specific timestamp expired.
- The editor attempting the update does not own the lock.

**Response Structure:**
```json
{
    "type": "update-failure",
    "data": {
        "timestamp": "12345",  // The TS the editor attempted to update
        "error": "Update rejected due to expired lock or lack of ownership." // Reason for failure
    }
}
```

This tells the editor that the update was rejected and they need to resolve the issue (e.g., re-lock the TS).

---

### Miscellaneous Notes
- **`sessionIdA`**: Represents the ID of the editor/session making the action. Replace this with the accurate editor/session identifier in your system.
- **Broadcast Messages**: 
  - Some messages (`lock-acquired`, `lock-released`, `update`) are sent to **all connected editors** in the stream.
  - Failure messages (`lock-failure`, `update-failure`) are only sent to the editor who initiated the action.

---

### **Frontend Integration** (Implementation Guide)

#### **1. Triggering Requests**
When frontend actions are performed, send the appropriate WebSocket message to the server using the specified `action`:
- **Lock**: Send `lock` with the `timestamp` to request a lock.
- **Unlock**: Send `unlock` with the `timestamp` to release a lock.
- **Update**: Send `update` with a list of changes (`timestamp` and `text`).

---

#### **2. Handling Responses**
Listen for the following message types and handle them accordingly:

| **Message Type**      | **Frontend Action**                                                                                     |
|------------------------|-------------------------------------------------------------------------------------------------------|
| `lock-acquired`        | Highlight the locked `timestamp` in the UI for the editor who acquired the lock.                      |
| `lock-failure`         | Show an error notification to the editor attempting to lock the `timestamp`.                          |
| `lock-released`        | Enable the `timestamp` for editing for all editors.                                                   |
| `update`               | Apply the updated text for the corresponding `timestamp` in the UI for all connected editors.         |
| `update-failure`       | Show an error notification to the editor attempting the update and disable further input for that TS. |

---

### Example Workflow

#### **Scenario 1: Lock and Update**

1. **Editor A Requests to Lock TS `12345`.**
   - Sends:
     ```json
     {
         "action": "lock",
         "data": {
             "timestamp": "12345"
         }
     }
     ```
   - Receives (on success):
     ```json
     {
         "type": "lock-acquired",
         "data": {
             "timestamp": "12345",
             "editor": "sessionIdA"
         }
     }
     ```

2. **Editor A Updates TS `12345`.**
   - Sends:
     ```json
     {
         "action": "update",
         "data": {
             "changes": [
                 {
                     "timestamp": "12345",
                     "text": "Updated text"
                 }
             ]
         }
     }
     ```
   - Broadcasts (to all editors):
     ```json
     {
         "type": "update",
         "data": [
             {
                 "timestamp": "12345",
                 "text": "Updated text",
                 "editor": "sessionIdA"
             }
         ]
     }
     ```

---

#### **Scenario 2: Lock Failure**

1. **Editor B Requests to Lock Already Locked TS `12345`.**
   - Sends:
     ```json
     {
         "action": "lock",
         "data": {
             "timestamp": "12345"
         }
     }
     ```
   - Receives:
     ```json
     {
         "type": "lock-failure",
         "data": {
             "timestamp": "12345",
             "error": "Lock not available or already held by another editor."
         }
     }
     ```

---

#### **Scenario 3: Update Failure**
1. **Editor A Tries to Update TS `12345` After the Lock Expires.**
   - Sends:
     ```json
     {
         "action": "update",
         "data": {
             "changes": [
                 {
                     "timestamp": "12345",
                     "text": "Update after lock expiry."
                 }
             ]
         }
     }
     ```
   - Receives (only for Editor A):
     ```json
     {
         "type": "update-failure",
         "data": {
             "timestamp": "12345",
             "error": "Update rejected due to expired lock or lack of ownership."
         }
     }
     ```

