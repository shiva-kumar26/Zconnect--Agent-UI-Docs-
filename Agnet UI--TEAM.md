# Team Filtering Implementation - Complete Documentation

## 📋 Table of Contents
1. [Overview](#overview)
2. [Problem Statement](#problem-statement)
3. [Solution Architecture](#solution-architecture)
4. [Changes Made](#changes-made)
5. [File-by-File Guide](#file-by-file-guide)
6. [Database Changes](#database-changes)
7. [Testing & Verification](#testing--verification)
8. [How It Works - Step by Step](#how-it-works---step-by-step)

---

## Overview

This document details the implementation of **Team Filtering** in the Zonius Contact Center application. The feature ensures that when an agent logs in, they only see their supervisor and team members, not all agents in the system.

**Key Achievement:** Agents are now filtered by supervisor hierarchy instead of viewing the entire agent list.

---

## Problem Statement

### Before Implementation:
- ❌ All agents could see ALL agents in the system on the Team page
- ❌ No supervisor-based filtering
- ❌ No team member isolation
- ❌ Extension data was missing from responses

### After Implementation:
- ✅ Agents see ONLY their supervisor + team members
- ✅ Supervisor-based hierarchical filtering works
- ✅ Team members are properly isolated
- ✅ Complete agent data (extension, firstname, lastname) is returned

---

## Solution Architecture

### Data Flow Diagram:

```
┌─────────────────────────────────────────────────────────────┐
│ 1. USER LOGS IN (Suresh - Extension 1037)                   │
│    Auth Context stores: auth.userName = "Suresh"            │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. FRONTEND - TeamPage Component Loads                       │
│    - Calls GET /api/agents with header X-User-ID: Suresh    │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. BACKEND - Route Handler                                   │
│    routes/agent_team_routes.py                              │
│    - Receives user_id from X-User-ID header                 │
│    - Calls get_agents_by_supervisor(db, "Suresh")           │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│ 4. BACKEND - Business Logic (CRUD Function)                 │
│    controller/agent_team_crud.py                            │
│    - Finds Suresh in DirectorySearch table                  │
│    - Reads supervisor_reference = "SupervisorThree"         │
│    - Finds SupervisorThree in database                      │
│    - Finds all agents with supervisor_reference =           │
│      "SupervisorThree"                                       │
│    - Returns: [SupervisorThree, Suresh, Agent7, Agent8,...] │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│ 5. FRONTEND - Display Results                                │
│    - Receives filtered agent list                           │
│    - Displays cards: Supervisor + Team Members              │
│    - Shows extension, name, status, team info               │
└─────────────────────────────────────────────────────────────┘
```

---

## Changes Made

### Summary Table:

| Component | File | Change Type | Details |
|-----------|------|-------------|---------|
| **Backend** | `controller/agent_team_crud.py` | New Function | Added `get_agents_by_supervisor()` |
| **Backend** | `routes/agent_team_routes.py` | Modified Endpoint | Updated to accept `X-User-ID` header |
| **Backend** | `models/models.py` | Updated Model | Added fields to `GetAllAgentsResponse` |
| **Frontend** | `src/config/config.js` | Config Fix | Changed endpoint from `/api/directory_search` to `/api/agents` |
| **Frontend** | `src/pages/TeamPage.jsx` | New Logic | Added header + removed local filtering |
| **Database** | `DirectorySearch` table | Data Cleanup | Fixed `supervisor_reference` capitalization |

---

## File-by-File Guide

---

### 1. Backend: `controller/agent_team_crud.py`

#### What Changed:
Added a new function `get_agents_by_supervisor()` that filters agents by supervisor hierarchy.

#### Complete Function Code:

```python
def get_agents_by_supervisor(db: Session, user_id: str) -> List[GetAllAgentsResponse]:
    """
    Get agents filtered by the logged-in user's supervisor
    Returns: supervisor + all agents under that supervisor
    """
    from models.User import DirectorySearch
    
    # Find the logged-in user
    current_user = db.query(DirectorySearch).filter(
        DirectorySearch.user_id == user_id
    ).first()
    
    if not current_user:
        logger.warning(f"User {user_id} not found in DirectorySearch")
        return []
    
    # Check if user is a supervisor
    if current_user.role and 'Supervisor' in current_user.role:
        # User is a supervisor - get their agents
        supervisor_user_id = user_id
    else:
        # User is an agent - get their supervisor
        supervisor_ref = current_user.supervisor_reference
        
        if not supervisor_ref or supervisor_ref.strip() == "":
            # No supervisor - return only current user
            logger.info(f"User {user_id} has no supervisor")
            return [GetAllAgentsResponse(
                agent_id=current_user.directory_id,
                agent_name=f"{current_user.firstname} {current_user.lastname}",
                team_id=0,
                team_name="No Team",
                extension=current_user.extension,
                firstname=current_user.firstname,
                lastname=current_user.lastname
            )]
        
        # supervisor_reference is a string
        supervisor_user_id = supervisor_ref
    
    # Find supervisor
    supervisor = db.query(DirectorySearch).filter(
        DirectorySearch.user_id == supervisor_user_id
    ).first()
    
    if not supervisor:
        logger.warning(f"Supervisor {supervisor_user_id} not found")
        return []
    
    # Get all agents with this supervisor (case-insensitive string match)
    agents_with_same_supervisor = db.query(DirectorySearch).filter(
        DirectorySearch.supervisor_reference.ilike(supervisor_user_id)
    ).all()
    
    result = []
    
    # Add supervisor first
    result.append(GetAllAgentsResponse(
        agent_id=supervisor.directory_id,
        agent_name=f"{supervisor.firstname} {supervisor.lastname}",
        team_id=0,
        team_name="Supervisor",
        extension=supervisor.extension,
        firstname=supervisor.firstname,
        lastname=supervisor.lastname
    ))
    
    # Add all agents
    for agent in agents_with_same_supervisor:
        result.append(GetAllAgentsResponse(
            agent_id=agent.directory_id,
            agent_name=f"{agent.firstname} {agent.lastname}",
            team_id=0,
            team_name="Team Member",
            extension=agent.extension,
            firstname=agent.firstname,
            lastname=agent.lastname
        ))
    
    logger.info(f"Found {len(result)} people for supervisor {supervisor_user_id}")
    return result
```

#### Key Logic:
1. **Finds current user** by `user_id`
2. **Checks if user is supervisor** - if yes, use their ID directly
3. **If agent**: reads their `supervisor_reference` field
4. **Finds supervisor** in database
5. **Finds all agents** under that supervisor using `.ilike()` for case-insensitive match
6. **Returns list** with supervisor first, then team members

---

### 2. Backend: `routes/agent_team_routes.py`

#### What Changed:
Updated the `/agents` endpoint to accept `X-User-ID` header and call the new filtering function.

#### Complete Route Code:

```python
@router.get("/agents", response_model=List[GetAllAgentsResponse])
def get_all_agents_handler(
    db: Session = Depends(get_db),
    user_id: Optional[str] = Header(None, alias="X-User-ID")
):
    import logging
    logger = logging.getLogger(__name__)
    
    logger.info(f"========================================")
    logger.info(f"🔍 AGENTS API CALLED")
    logger.info(f"🔍 RECEIVED user_id: {user_id}")
    logger.info(f"========================================")
    
    if not user_id:
        logger.error("❌ No user_id in header!")
        raise HTTPException(status_code=401, detail="User ID not provided in headers")
    
    agents = agent_team_crud.get_agents_by_supervisor(db, user_id)
    
    logger.info(f"✅ RETURNING {len(agents)} agents")
    logger.info(f"========================================")
    
    if not agents:
        raise HTTPException(status_code=404, detail="No agents found for this user")
    
    return agents
```

#### Key Changes:
- **Added parameter:** `user_id: Optional[str] = Header(None, alias="X-User-ID")`
  - This reads the `X-User-ID` header from the request
- **Changed function call:** From `get_all_agents()` to `get_agents_by_supervisor(db, user_id)`
- **Added validation:** Checks if `user_id` header is present
- **Added logging:** Helps debug and track API calls

---

### 3. Backend: `main.py`

#### What Changed:
The router was already registered. No changes needed if it was done correctly initially.

#### Verification:
```python
app.include_router(agent_team_router, prefix="/api", tags=["Agent Team"])
```

This line should exist in your `main.py` file around line 107.

---

### 4. Backend: `models/models.py`

#### What Changed:
Updated `GetAllAgentsResponse` model to include new fields.

#### Before:
```python
class GetAllAgentsResponse(BaseModel):
    agent_id: int
    agent_name: str  
    team_id: int
    team_name: str
```

#### After:
```python
class GetAllAgentsResponse(BaseModel):
    agent_id: int
    agent_name: str  
    team_id: int
    team_name: str
    extension: Optional[str] = None
    firstname: Optional[str] = None
    lastname: Optional[str] = None
    
    class Config:
        orm_mode = True
```

#### Why:
- **extension**: Shows agent's extension number (1031, 1035, etc.)
- **firstname**: Agent's first name
- **lastname**: Agent's last name
- **orm_mode**: Allows automatic conversion from ORM objects to Pydantic models

---

### 5. Frontend: `src/config/config.js`

#### What Changed:
Fixed the agents endpoint URL.

#### Before:
```javascript
agents: "/api/directory_search",
```

#### After:
```javascript
agents: "/api/agents",
```

#### Why:
- The old endpoint was fetching ALL agents from `directory_search`
- The new endpoint calls our filtered `/api/agents` endpoint that respects the supervisor hierarchy

---

### 6. Frontend: `src/pages/TeamPage.jsx`

#### What Changed:
Modified the data loading logic to send the user's ID to the backend.

#### Complete Updated useEffect:

```javascript
useEffect(() => {
  const loadData = async () => {
    try {
      const response = await axios.get(
        `${backendConfig.baseURL}${backendConfig.agents}`,
        {
          headers: {
            'X-User-ID': auth?.userName  // ← SEND USER ID IN HEADER
          }
        }
      );
      
      console.log("Team members from backend:", response.data);
      setTeamMembers(response.data);
    } catch (error) {
      console.error("Error loading team members:", error);
    }
  };

  if (auth?.userName) {  // ← ONLY LOAD IF LOGGED IN
    loadData();
  }
}, [auth?.userName]);  // ← TRIGGER WHEN USERNAME CHANGES
```

#### What Was Removed:
```javascript
// ❌ REMOVED: This entire filtering logic
useEffect(() => {
  if (teamMembers.length === 0 || !auth?.userName) return;

  const currentAgent = teamMembers.find(
    (member) => member.user_id === auth.userName || member.extension === auth.extension
  );
  
  // ... filtering logic ...
  
  setFilteredTeamMembers(filtered);
}, [teamMembers, auth]);
```

#### Why:
- Backend now does the filtering, no need for frontend to do it
- Simpler, cleaner code
- Single source of truth (backend database)

#### What Was Deleted:
- ❌ Removed: `const [filteredTeamMembers, setFilteredTeamMembers] = useState([]);`
- ❌ Removed: Changed `.map` from `filteredTeamMembers` to `teamMembers`

---

### 7. Database: `DirectorySearch` table

#### What Changed:
Fixed inconsistent capitalization in `supervisor_reference` field.

#### SQL Fix:
```sql
-- Fix all supervisor references to match correct capitalization
UPDATE directory_search 
SET supervisor_reference = 'SupervisorOne' 
WHERE supervisor_reference = 'Supervisorone';

UPDATE directory_search 
SET supervisor_reference = 'SupervisorTwo' 
WHERE supervisor_reference = 'Supervisortwo';

UPDATE directory_search 
SET supervisor_reference = 'SupervisorThree' 
WHERE supervisor_reference = 'Supervisorthree';
```

#### Before:
```
Supervisor reference values:
- "Supervisorone" (lowercase 'o')
- "Supervisortwo" (lowercase 't')
- "Supervisorthree" (lowercase 't')
```

#### After:
```
Supervisor reference values:
- "SupervisorOne" (proper case)
- "SupervisorTwo" (proper case)
- "SupervisorThree" (proper case)
```

#### Why:
- Backend query looks for exact matches
- Case-sensitive string comparison failed with wrong capitalization
- Fixing the database ensures consistency

---

## How It Works - Step by Step

### Scenario: Suresh (Extension 1037) Logs In

#### Step 1: Authentication
```
User: Suresh (Extension 1037)
User ID stored in auth context: "Suresh"
```

#### Step 2: Frontend Loads Team Page
```javascript
TeamPage.jsx useEffect triggers:
- Checks: auth?.userName exists? YES → "Suresh"
- Creates axios request with header:
  GET https://10.16.7.96/api/agents
  Headers: {
    'X-User-ID': 'Suresh'
  }
```

#### Step 3: Backend Route Receives Request
```python
Route: GET /api/agents
Header received: X-User-ID = "Suresh"
Calls: get_agents_by_supervisor(db, "Suresh")
```

#### Step 4: Backend Queries Database
```sql
1. SELECT * FROM directory_search WHERE user_id = 'Suresh'
   Result: Found Suresh (Extension 1037)
   
2. Check: Is Suresh a supervisor? NO
   Read supervisor_reference: "SupervisorThree"
   
3. SELECT * FROM directory_search WHERE user_id = 'SupervisorThree'
   Result: Found SupervisorThree
   
4. SELECT * FROM directory_search 
   WHERE supervisor_reference ILIKE 'SupervisorThree'
   Result: [Suresh, Supraja, ...]
```

#### Step 5: Backend Returns Response
```json
[
  {
    "agent_id": 40,
    "agent_name": "Supervisor Three",
    "team_id": 0,
    "team_name": "Supervisor",
    "extension": "1040",
    "firstname": "Supervisor",
    "lastname": "Three"
  },
  {
    "agent_id": 47,
    "agent_name": "Agent Seven",
    "team_id": 0,
    "team_name": "Team Member",
    "extension": "1037",
    "firstname": "Suresh",
    "lastname": "..."
  },
  // ... more team members ...
]
```

#### Step 6: Frontend Displays Results
```javascript
setTeamMembers(response.data)
  ↓
Renders cards for:
- SupervisorThree (with "Supervisor" label)
- Agent Seven (with "Team Member" label)
- Agent Eight
- Agent Nine
```

---

## Testing & Verification

### Test Cases:

#### Test 1: Login as Agent (Suresh)
```
Expected: See SupervisorThree + team members
Result: ✅ PASS
```

#### Test 2: Login as Different Agent (Durga)
```
Expected: See SupervisorOne + team members
Result: ✅ PASS
```

#### Test 3: Login as Supervisor (SupervisorThree)
```
Expected: See own profile + all agents under them
Result: ✅ PASS
```

#### Test 4: Check Extensions Display
```
Expected: Extensions visible (1037, 1040, etc.)
Result: ✅ PASS (After model update)
```

### Backend Logs to Verify:
```
✅ AGENTS API CALLED
✅ RECEIVED user_id: Suresh
✅ RETURNING 4 agents
```

---

## Troubleshooting Guide

### Issue 1: Getting 404 "No agents found"
**Cause:** Supervisor not found in database
**Solution:** Check `supervisor_reference` capitalization matches actual supervisor's `user_id`

### Issue 2: Extensions not showing
**Cause:** `GetAllAgentsResponse` model missing fields
**Solution:** Add `extension`, `firstname`, `lastname` to model

### Issue 3: Seeing all agents instead of filtered
**Cause:** Frontend still calling old endpoint
**Solution:** Update `config.js` to use `/api/agents` instead of `/api/directory_search`

### Issue 4: Header not being sent
**Cause:** `auth.userName` is undefined
**Solution:** Check login response includes `userName` field

---

## Summary of Implementation

### Files Modified: 6
- ✅ `controller/agent_team_crud.py` - Added filtering function
- ✅ `routes/agent_team_routes.py` - Updated endpoint
- ✅ `models/models.py` - Extended response model
- ✅ `src/config/config.js` - Fixed endpoint URL
- ✅ `src/pages/TeamPage.jsx` - Updated data loading
- ✅ `DirectorySearch` table - Fixed data

### Key Concepts:
1. **Supervisor Hierarchy** - Agents grouped under supervisors
2. **Header-Based Filtering** - `X-User-ID` header identifies current user
3. **Database Query Optimization** - Uses `.ilike()` for case-insensitive matching
4. **Backend Processing** - All filtering done server-side (more secure)
5. **Type Safety** - Pydantic models ensure correct data format

### Flow Summary:
```
User Login
    ↓
Frontend sends X-User-ID header
    ↓
Backend receives user ID
    ↓
Database query: Find supervisor
    ↓
Database query: Find team members
    ↓
Backend returns filtered list
    ↓
Frontend displays team
```

---

## Deployment Checklist

- [x] Updated `controller/agent_team_crud.py`
- [x] Updated `routes/agent_team_routes.py`
- [x] Updated `models/models.py`
- [x] Updated `src/config/config.js`
- [x] Updated `src/pages/TeamPage.jsx`
- [x] Fixed database supervisor_reference values
- [x] Restarted backend service
- [x] Hard-refreshed frontend (Ctrl + Shift + R)
- [x] Tested with multiple users
- [x] Verified extensions display correctly
- [x] Checked backend logs
- [x] Verified filtering works correctly

---

## Conclusion

The Team Filtering feature has been successfully implemented! Agents now see only their supervisor and team members instead of all agents in the system. This improves security, usability, and provides a cleaner interface for team management.

For any questions or issues, refer to the troubleshooting section or check the backend logs using:
```bash
sudo journalctl -u fastapi.service -f
```